---
title: "Storage — PV / PVC / StorageClass"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:46:00+09:00
tags: [devops, kubernetes, storage]
---

# Storage — PV / PVC / StorageClass

**[[kubernetes|↑ k8s]]**

---

## 1. 3 가지

| | 무엇 |
| --- | --- |
| **PV** (PersistentVolume) | cluster 의 storage 자원 (admin 의 추상) |
| **PVC** (PersistentVolumeClaim) | 앱이 요청 (size + access mode) |
| **StorageClass** | dynamic 생성 — cloud 가 자동 PV 만듦 |

---

## 2. StorageClass (cloud 자동)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: gp3}
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete       # 또는 Retain (data 보존)
```

---

## 3. PVC (앱에서 요청)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: pg-data}
spec:
  storageClassName: gp3
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
```

---

## 4. Pod 에 mount

```yaml
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pg-data
```

---

## 5. Access mode

| Mode | 의미 |
| --- | --- |
| **ReadWriteOnce (RWO)** ★ | 1 노드 만 r/w (EBS / GCP PD) |
| ReadOnlyMany (ROX) | 다중 노드 read |
| ReadWriteMany (RWX) | 다중 노드 r/w (EFS / Filestore) |
| ReadWriteOncePod (RWOP) | 1 Pod 만 (k8s 1.22+) |

---

## 6. StatefulSet (DB)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: postgres}
spec:
  serviceName: postgres
  replicas: 1
  selector: {matchLabels: {app: postgres}}
  volumeClaimTemplates:
    - metadata: {name: data}
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources: {requests: {storage: 20Gi}}
  template: ...
```

→ Pod 마다 자기 PVC 자동.

---

## 7. 함정

1. **DB Deployment 사용** (StatefulSet X) → Pod 이동 시 다른 데이터.
2. **RWX 가정** (RWO 실제) → 다중 Pod 충돌.
3. **reclaimPolicy=Delete + 실수 PVC delete** → 영구 손실 → Retain.
4. **VolumeBindingMode Immediate** → 노드 fail 시 fail (cross-AZ).
5. **storage 적게** → 가득 차면 늘릴 수 있지만 downtime.

---

## 8. 관련

- [[kubernetes|↑ k8s]]
- [[concepts]]
