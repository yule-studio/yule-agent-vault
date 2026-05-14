---
title: "GCP Persistent Disk"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:32:00+09:00
tags:
  - gcp
  - storage
  - persistent-disk
---

# GCP Persistent Disk

**[[storage|↑ Storage]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

GCE 의 **block storage** — AWS EBS 동등. **online resize**, **snapshot**, **regional replication** 가능.

---

## 2. Type

| Type | IOPS | Throughput | $/GB·월 |
| --- | --- | --- | --- |
| **pd-standard** | HDD | - | $0.040 |
| **pd-balanced** | 균형 SSD | 6K | $0.10 |
| **pd-ssd** | 빠른 SSD | 30K | $0.17 |
| **pd-extreme** | 극한 | 120K | $0.125 + IOPS |
| **Hyperdisk Balanced/Throughput/Extreme** | IOPS/처리량 독립 조정 | 가변 | 가변 |

→ 기본 = pd-balanced. DB = pd-ssd. 거대 DB = hyperdisk.

---

## 3. 설치 / 사용

```bash
gcloud compute disks create data --type=pd-ssd --size=100GB --zone=asia-northeast3-a
gcloud compute instances attach-disk myapp --disk=data --zone=asia-northeast3-a

# 인스턴스 안에서
sudo mkfs.xfs /dev/sdb
sudo mkdir /data && sudo mount /dev/sdb /data
```

---

## 4. Snapshot

```bash
gcloud compute disks snapshot data --snapshot-names=data-2026-05-14
gcloud compute snapshots list

# 자동 — Snapshot Schedule
gcloud compute resource-policies create snapshot-schedule daily \
  --region=asia-northeast3 \
  --max-retention-days=14 \
  --daily-schedule \
  --start-time=18:00

gcloud compute disks add-resource-policies data --resource-policies=daily
```

- incremental
- cross-region copy 가능
- Standard storage 에 저장

---

## 5. Regional Disk

```bash
gcloud compute disks create data \
  --type=pd-ssd \
  --size=100GB \
  --region=asia-northeast3 \
  --replica-zones=asia-northeast3-a,asia-northeast3-b
```

→ 2 zone 동시 replication. zone 장애 시 다른 zone 으로 attach 가능.

---

## 6. Online Resize

```bash
gcloud compute disks resize data --size=200GB --zone=asia-northeast3-a

# 인스턴스 안에서
sudo growpart /dev/sdb 1
sudo resize2fs /dev/sdb1     # ext4
# 또는 sudo xfs_growfs /data
```

→ 다운타임 X.

---

## 7. 함정

- snapshot retention 누락 → 누적
- regional disk = 2x 비용
- zone bind — 다른 zone 으로는 snapshot 통해
- max IOPS = 인스턴스 type 한계

---

## 8. 관련

- [[storage]]
- [[cloud-storage]] — object
- [[../compute/gce]]
