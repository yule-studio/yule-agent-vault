---
title: "OCI Block Volume"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:35:00+09:00
tags:
  - oci
  - storage
  - block-volume
---

# OCI Block Volume

**[[storage|↑ Storage]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 block storage (EBS / Persistent Disk 동등). VM 에 attach. 200 GB Always Free.

---

## 2. Performance Tier

| Tier | VPU/GB | 의미 |
| --- | --- | --- |
| **Lower Cost** | 0 | HDD-like |
| **Balanced** | 10 | 기본 |
| **Higher Performance** | 20 | SSD high |
| **Ultra High** | 30-120 | 최대 IOPS |

→ VPU = Volume Performance Units. 동적 변경 가능 (online).

---

## 3. 사용

```hcl
resource "oci_core_volume" "data" {
  compartment_id      = var.compartment_ocid
  availability_domain = data.oci_identity_availability_domain.ad.name
  display_name        = "myapp-data"
  size_in_gbs         = 100
  vpus_per_gb         = 10                  # Balanced
  kms_key_id          = oci_kms_key.disk.id
}

resource "oci_core_volume_attachment" "att" {
  attachment_type = "paravirtualized"        # 또는 iscsi
  instance_id     = oci_core_instance.vm.id
  volume_id       = oci_core_volume.data.id
}
```

---

## 4. Mount (Linux)

```bash
# /dev/sdb 또는 iSCSI 의 device
sudo mkfs.xfs /dev/sdb
sudo mkdir /data
sudo mount /dev/sdb /data
echo "/dev/sdb /data xfs defaults,_netdev,nofail 0 2" | sudo tee -a /etc/fstab
```

---

## 5. Snapshot / Clone / Backup

```bash
oci bv volume-backup create --volume-id <id> --display-name daily-backup --type INCREMENTAL
oci bv volume-backup-policy-assignment create-volume-backup-policy-assignment \
  --asset-id <volume> --policy-id <policy>           # automated
```

- Backup = cross-region 가능
- Clone = 같은 region 즉시 복사 (writable)

---

## 6. 가격

```
Storage:    $0.0255 / GB·월 (Balanced)
VPU:        $0.0017 / GB·월 추가 (10 VPU 기본)
Backup:     $0.0204 / GB·월 (Object Storage 안)
Free tier:  200 GB block (object 와 합)
```

---

## 7. 함정

- iSCSI = manual attach script 필요 (paravirtualized 권장)
- size 늘려도 filesystem resize 별도 (`xfs_growfs`)
- VPU 동적 변경 OK, 그러나 일시적 latency
- AD-local — 같은 AD 의 VM 만 (Seoul = 1 AD 라 무관)
- snapshot 은 enterprise level 만 (옛, 변경 가능)

---

## 8. 관련

- [[storage]]
- [[object-storage]]
- [[../security/vault]]
