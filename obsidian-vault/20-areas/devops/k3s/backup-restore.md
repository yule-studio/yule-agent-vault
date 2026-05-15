---
title: "k3s backup / restore — etcd snapshot / SQLite / external DB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:13:00+09:00
tags: [devops, k3s, backup, dr]
---

# k3s backup / restore — etcd snapshot / SQLite / external DB

**[[k3s|↑ k3s]]**

---

## 1. 무엇을 백업

```
1. cluster state — etcd / SQLite / external DB
2. manifests — /var/lib/rancher/k3s/server/manifests/
3. config — /etc/rancher/k3s/
4. data — PV (별도, Velero / Longhorn)
5. secrets — Vault / Sealed / ESO 외부 동기화면 OK
```

→ 이 모두를 자동화해야 진짜 DR.

---

## 2. embedded etcd snapshot (★ default 활성)

```yaml
# /etc/rancher/k3s/config.yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"     # 6시간마다
etcd-snapshot-retention: 24                     # 24개 유지 (6시간 × 24 = 6일)
etcd-snapshot-dir: /var/lib/rancher/k3s/server/db/snapshots
# S3 직접 upload (★)
etcd-s3: true
etcd-s3-endpoint: s3.amazonaws.com
etcd-s3-region: ap-northeast-2
etcd-s3-bucket: my-k3s-backup
etcd-s3-access-key: AKIA...
etcd-s3-secret-key: ...
etcd-s3-folder: k3s-prod
```

```bash
# 수동 snapshot
sudo k3s etcd-snapshot save --name pre-upgrade

# 목록
sudo k3s etcd-snapshot ls
```

→ default 매 12시간. **production = 자주 + S3 upload**.

---

## 3. embedded etcd restore (★ 위기)

```bash
# 모든 server stop
sudo systemctl stop k3s     # 각 server 에서

# restore (한 server 에서만)
sudo k3s server \
    --cluster-reset \
    --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/on-demand-server1-1715740800.zip

# 또는 S3
sudo k3s server \
    --cluster-reset \
    --etcd-s3 \
    --etcd-s3-endpoint=s3.amazonaws.com \
    --cluster-reset-restore-path=on-demand-server1-...

# restore 완료 → 종료 → service start
sudo systemctl start k3s

# 다른 server 들 join (--cluster-init 의 새 cluster 로 인식하므로 데이터 제거 후 join)
ssh server2
sudo systemctl stop k3s
sudo rm -rf /var/lib/rancher/k3s/server/db
sudo systemctl start k3s
```

→ 검증 절차 — 정기 drill 필수.

---

## 4. SQLite backup (single node)

```bash
# 파일 1개 복사
sudo cp /var/lib/rancher/k3s/server/db/state.db ./backup-$(date +%Y%m%d).db

# 또는 sqlite3 backup 명령 (live backup)
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db ".backup ./backup.db"

# 정기 cron
0 */6 * * * sqlite3 /var/lib/rancher/k3s/server/db/state.db ".backup /backup/k3s-$(date +\%Y\%m\%d-\%H).db"
```

복원:
```bash
sudo systemctl stop k3s
sudo cp ./backup.db /var/lib/rancher/k3s/server/db/state.db
sudo systemctl start k3s
```

→ k3s stop 필수 (live restore X).

---

## 5. external DB backup

```bash
# Postgres
pg_dump --format=custom k3sdb > k3s-$(date +%Y%m%d).dump
aws s3 cp k3s-*.dump s3://my-backup/

# 복원
pg_restore --clean --dbname=k3sdb k3s-20260515.dump

# RDS 등 managed = 자동 backup + snapshot
```

→ external DB 의 장점 = DB 의 native backup 사용.

---

## 6. manifests / config 백업

```bash
# 일별 backup 스크립트
sudo tar -czf /backup/k3s-config-$(date +%Y%m%d).tar.gz \
    /etc/rancher/k3s/ \
    /var/lib/rancher/k3s/server/manifests/ \
    /var/lib/rancher/k3s/server/token

aws s3 cp /backup/*.tar.gz s3://my-backup/config/
```

→ 새 server 띄울 때 즉시 복원 가능.

---

## 7. Velero (★ workload backup)

```bash
helm install velero vmware-tanzu/velero \
    -n velero --create-namespace \
    --set configuration.provider=aws \
    --set configuration.backupStorageLocation[0].bucket=my-velero-backup \
    --set configuration.backupStorageLocation[0].config.region=ap-northeast-2 \
    --set initContainers[0].name=velero-plugin-for-aws \
    --set initContainers[0].image=velero/velero-plugin-for-aws:v1.8.0 \
    --set initContainers[0].volumeMounts[0].mountPath=/target \
    --set initContainers[0].volumeMounts[0].name=plugins
```

```bash
# 정기 백업 (namespace 단위)
velero schedule create daily \
    --schedule "0 1 * * *" \
    --include-namespaces=prod,monitoring \
    --ttl 720h0m0s \
    --include-cluster-resources=true

# 즉시 backup
velero backup create pre-migration \
    --include-namespaces=prod

# restore
velero restore create --from-backup pre-migration
```

→ namespace + manifests + PV 까지 백업 (CSI snapshot 통합).

---

## 8. Longhorn 자체 backup

```yaml
# recurring job
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata: {name: daily-backup, namespace: longhorn-system}
spec:
  cron: "0 2 * * *"
  task: "backup"
  groups: ["default"]
  retain: 30
  concurrency: 2
```

```yaml
# BackupTarget — S3
apiVersion: longhorn.io/v1beta2
kind: BackupTarget
metadata: {name: default, namespace: longhorn-system}
spec:
  backupTargetURL: "s3://my-longhorn-backup@us-east-1/"
  credentialSecret: aws-creds
  pollInterval: 5m
```

→ snapshot + backup 분리. snapshot = 빠름 / local, backup = S3.

---

## 9. backup 검증 (★ 안 한 백업 = 가치 0)

```bash
# 1. 정기 (월 1회) — 다른 cluster 에 복원 시도
# 2. checksum / size 검증
# 3. velero 의 backup describe 로 status PartialFailed 점검
# 4. drill 후 보고서

velero backup describe daily-20260515 --details
velero backup logs daily-20260515
```

---

## 10. RTO / RPO 목표

```
RTO (복구 시간 목표):
  k3s control plane: 30분
  workload (Velero restore): 2시간

RPO (데이터 손실 목표):
  etcd snapshot: 6시간
  PV (Longhorn): 1시간
  external DB: 1시간 (point-in-time)
```

→ 짧을수록 비용. business 와 합의.

---

## 11. DR drill (★ 분기)

```
1. 새 cluster 에 backup 복원
2. workload reapply
3. external service (DB / 외부 API) 연결
4. smoke test
5. RTO / RPO 실측
6. 문제점 정리 → action item
```

→ 안 하면 실제 disaster 시 backup 동작 안 함.

---

## 12. 함정

1. **snapshot 만, S3 미upload** — server disk 손실 시 잃음.
2. **manifest backup 안 함** — 새 cluster 띄워도 service 자동 안 옴.
3. **secrets 분리 X** — backup 에 노출.
4. **schedule 만, retention 없음** — disk full.
5. **restore drill 안 함** — 실제 disaster 시 fail.
6. **PV backup 빠짐** — etcd 만 백업 + DB 없음.
7. **encryption 없음** — backup 도난 시 cluster 평문.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[ha-mode]]
- [[upgrade-strategy]]
- [[../sre/disaster-recovery|↗ DR]]
