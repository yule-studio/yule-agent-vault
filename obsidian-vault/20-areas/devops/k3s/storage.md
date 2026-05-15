---
title: "k3s storage — local-path / Longhorn / external CSI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:09:00+09:00
tags: [devops, k3s, storage, csi]
---

# k3s storage — local-path / Longhorn / external CSI

**[[k3s|↑ k3s]]**

---

## 1. 선택지

| | 용도 | 강점 | 약점 |
| --- | --- | --- | --- |
| **local-path** (default) | dev / single node | 즉시 사용, 빠름 | node-pinned (pod 가 그 node 만) |
| **Longhorn** | HA / replication | block storage, snapshot, cross-node | resource 부담 |
| **OpenEBS** | flexible | LocalPV + replication | complexity |
| **Ceph (Rook)** | enterprise | 강력 | 큰 cluster + 운영 부담 |
| **NFS** | shared file | 간단 | latency, NFS server SPOF |
| **CSI driver (cloud)** | EBS / GCE-PD / Azure Disk | managed | cloud 종속 |
| **TopoLVM** | host LVM | thin-provisioning | LVM 학습 |

---

## 2. local-path (default)

```yaml
# StorageClass 자동 (kube-system/local-path)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

사용:
```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources: {requests: {storage: 1Gi}}

---
# Pod
apiVersion: v1
kind: Pod
metadata: {name: app}
spec:
  containers:
    - name: app
      volumeMounts:
        - {mountPath: /data, name: data}
  volumes:
    - name: data
      persistentVolumeClaim: {claimName: data}
```

→ 실제 파일은 `/var/lib/rancher/k3s/storage/<pvc-name>` 에 생성.

→ pod 가 같은 node 에 reschedule 되어야 데이터 보임.

---

## 3. local-path 의 한계

```
1. node-pinned — node 떨어지면 데이터 접근 불가
2. backup 별도 필요 — k3s 가 안 해줌
3. replication 없음
4. RWO 만 (ReadWriteMany 안 됨)
5. node 의 disk full 시 다른 PVC 영향
```

→ dev / non-critical OK. production replicated workload 면 Longhorn 등.

---

## 4. Longhorn (★ 권장 — HA)

```bash
# 설치 (각 node 에 open-iscsi 필요)
sudo apt install -y open-iscsi nfs-common

helm repo add longhorn https://charts.longhorn.io
helm install longhorn longhorn/longhorn \
    -n longhorn-system --create-namespace
```

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: longhorn}
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
  recurringJobSelector: '[{"name":"daily-backup","isGroup":false}]'
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```yaml
# RWX (ReadWriteMany — share)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: longhorn-rwx}
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  shareManager: "true"
```

기능:
- block storage replicate (3 node 권장)
- snapshot / clone
- backup → S3 / NFS
- expand (PVC resize)
- RWO + RWX (NFS 통해)
- UI dashboard

→ on-prem k3s 의 표준.

---

## 5. NFS provisioner (간단 shared)

```bash
helm install nfs-subdir-external-provisioner \
    nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=nfs.internal \
    --set nfs.path=/exported/path
```

→ NFS server 가 외부에 있을 때. 가장 간단한 RWX.

→ 한계: NFS 자체 HA 부담, latency 큼.

---

## 6. cloud CSI (AWS EBS 예)

```bash
# k3s 의 cloud controller manager 활성
# config.yaml
disable-cloud-controller: false
kubelet-arg:
  - "cloud-provider=external"

# AWS CCM 설치
helm install aws-cloud-controller-manager \
    aws-cloud-controller-manager/aws-cloud-controller-manager

# AWS EBS CSI
helm install aws-ebs-csi-driver \
    aws-ebs-csi-driver/aws-ebs-csi-driver \
    -n kube-system
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: gp3}
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

→ k3s on EC2 시 EBS 사용 가능.

---

## 7. backup 전략

```
local-path:
  - tar / rsync 직접
  - velero (k8s backup 도구)
  - cron job 으로 정기 backup

Longhorn:
  - 내장 recurring backup → S3
  - snapshot 즉시 + scheduled

NFS:
  - NFS server backup

Cloud (EBS):
  - snapshot via DLM (Data Lifecycle Manager)
  - Velero 통합
```

---

## 8. Velero (★ k8s backup)

```bash
helm install velero vmware-tanzu/velero \
    -n velero --create-namespace \
    --set configuration.backupStorageLocation[0].bucket=my-k3s-backup \
    --set configuration.backupStorageLocation[0].provider=aws

# 백업
velero backup create daily --include-namespaces=prod --schedule="0 2 * * *"

# 복원
velero restore create --from-backup daily-20260515
```

→ namespace / resource / PV 전체 백업.

---

## 9. dynamic resize

```yaml
# PVC expand
spec:
  resources:
    requests:
      storage: 10Gi    # 1Gi → 10Gi

# StorageClass 에 allowVolumeExpansion: true 필수
```

→ Longhorn / EBS 지원. local-path 미지원 (수동 file 이동).

---

## 10. RWO vs RWX vs ROX

| | 무엇 |
| --- | --- |
| **RWO** (ReadWriteOnce) | 1 node 만 mount (DB 등) |
| **RWX** (ReadWriteMany) | 여러 node mount (공유 file) |
| **ROX** (ReadOnlyMany) | 여러 node 읽기만 |
| **RWOP** (ReadWriteOncePod) | 1 pod 만 (k8s 1.22+) |

→ DB / Postgres = RWO. 공유 upload / shared file = RWX (NFS / Longhorn RWX).

---

## 11. 함정

1. **local-path + 여러 replica** — pod 가 다른 node 가면 데이터 X.
2. **Longhorn 1 node** — replica 1 == HA 없음.
3. **open-iscsi 안 설치** — Longhorn 동작 안 함.
4. **NFS server SPOF** — 전체 service 영향.
5. **backup 검증 안 함** — 복구 시 깨짐.
6. **PVC 삭제 = data 삭제** — `reclaimPolicy: Retain` 고려.
7. **storage class default 변경** — 기존 manifest break.

---

## 12. 관련

- [[k3s|↑ k3s]]
- [[backup-restore]]
- [[../kubernetes/storage|↗ k8s storage]]
