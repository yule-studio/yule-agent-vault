---
title: "RAID 레벨"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, raid, parity, zfs, mdadm, ure]
---

# RAID 레벨

**[[storage|↑ 저장 장치]]**

> 파일 시스템 / LVM / ZFS 관점은 [[../../operating-system/operating-system]] 의 파일 시스템 절.

## 1. 표

| 레벨 | 디스크 수 | 손실 허용 | 가용 용량 | random read | random write |
| --- | --- | --- | --- | --- | --- |
| **RAID 0** | ≥2 | 0 | 100% | ↑↑ | ↑↑ |
| **RAID 1** | 2 | 1 | 50% | ↑ | 단일 디스크 수준 |
| **RAID 5** | ≥3 | 1 | (n-1)/n | ↑ | ↓ (parity) |
| **RAID 6** | ≥4 | 2 | (n-2)/n | ↑ | ↓↓ |
| **RAID 10** | ≥4 | n/2 (분산 운) | 50% | ↑↑ | ↑ |
| **RAID-Z2 / Z3 (ZFS)** | ≥4/5 | 2/3 | ≈(n-2 or 3)/n | ↑ | 보통 |

## 2. 동작 원리

### RAID 0 — Striping
- 데이터를 N 디스크에 분산. **중복성 0**.
- 어떤 디스크 1 개 fail 시 전체 손실.
- 빠르고 큰 임시 공간 (캐시) 용도 외에는 권장 안 함.

### RAID 1 — Mirroring
- 같은 데이터를 2 디스크에 복제.
- 1 디스크 fail 시 무사. read 는 양쪽 분산.
- 가용 용량 50%.

### RAID 5 — Distributed Parity
- N-1 디스크에 데이터 + 1 디스크 분량의 parity 를 N 디스크에 회전 분산.
- **1 디스크 fail 까지 복구 가능**.
- write 마다 parity 재계산 → write 성능 ↓.
- **rebuild 동안 read 부하 ↑ + URE 누적** → 큰 HDD 환경에서는 두 번째 디스크 fail 위험 ↑ (다음 절).

### RAID 6 — Dual Parity
- parity 2 종 (Reed-Solomon 다항식 기반) 을 분산.
- 2 디스크 fail 까지 복구.
- write 성능 RAID 5 보다 더 ↓ (parity 2 개 갱신).
- 큰 HDD 어레이의 현실적인 default.

### RAID 10 — Stripe of Mirrors
- 먼저 mirror 한 다음 stripe.
- 가용 용량 50%. 빠른 write + 좋은 read.
- **여러 디스크 fail 가능 (mirror pair 의 한 쪽씩만 죽으면)** — 운 좋으면 n/2 까지.

## 3. URE / rebuild 위험

**URE (Unrecoverable Read Error)** — HDD 사양상 평균 `1 in 10^14 bit` (consumer) ~ `1 in 10^16 bit` (enterprise) 빈도로 read 실패.

10 TB drive 1 개 = 80 × 10^12 bit. consumer 사양이면 평균 1 회 / 디스크 read 1 회 풀스캔.

→ 8 × 10 TB RAID 5 의 rebuild = 70 TB read = **URE 누적 확률이 위험 수준**.

대안:
- RAID 6 (2 fail 허용).
- ZFS RAID-Z2 / Z3 + 정기 scrub.
- RAID 10 (rebuild 가 한 mirror pair 만 영향).

## 4. ZFS / Btrfs 의 RAID-Z

ZFS 는 끝-끝 checksum 으로 bit rot 도 자동 정정. RAID-Z2 가 RAID 6 의 ZFS 버전.

- **scrub** — 주기적으로 모든 데이터 read + checksum 검증 + 손상 시 parity 로 복구. 일반 RAID 가 못 하는 silent corruption 방어.
- **vdev** — top-level pool 구성 단위. mirror / RAID-Z / single.

## 5. 하드웨어 RAID vs 소프트웨어 RAID

| 비교 | 하드웨어 RAID | 소프트웨어 RAID |
| --- | --- | --- |
| **컨트롤러** | 별도 ASIC + DRAM + BBU | OS 가 처리 (mdadm / LVM / ZFS / btrfs) |
| **OS 부담** | 적음 | 일부 CPU 사용 |
| **BBU / FBWC** | parity 캐시 안정 | 자체 안정 (ZFS sync write) |
| **벤더 고정** | 컨트롤러 dies → 같은 모델 필요 | 어떤 호스트로도 옮김 |
| **유연성** | RAID 5/6/10 정도 | 거의 무한 (Z3, dRAID, hybrid 등) |

현대 데이터센터는 보통 **HBA + ZFS / Ceph / RAID-Z** 로 소프트웨어 RAID 채택. 하드웨어 RAID 는 옛 enterprise 와 SAN 컨트롤러에 잔존.

## 6. 진단

```bash
# Linux mdadm (소프트웨어)
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --examine /dev/sd[a-d]

# ZFS
zpool status -v
zpool scrub tank
zpool list

# HW RAID (LSI / Broadcom MegaRAID)
storcli /c0 show
storcli /c0/v0 show
```

## 7. 함정

1. **RAID = 백업 아님** — 파일 삭제 / ransomware / 휴먼 에러는 RAID 가 막지 않음. 별도 백업 필수.
2. **RAID 5 + 큰 HDD** — rebuild URE 누적 → 두 번째 fail.
3. **rebuild 도중 사용자 IO 풀가동** — 더 길어지고 fail 확률 ↑. priority 낮춤.
4. **다른 제조사 / 펌웨어 디스크 혼용 in RAID** — 일부 펌웨어 차이가 stripe 의 latency variance 를 키움.
5. **온도 / vibration 무시** — 같은 batch 디스크가 동일 시점에 fail 하는 사례 흔함. spare pool + 정기 교체.

## 8. 관련

- [[storage]]
- [[hdd]]
- [[ssd-and-ftl]]
- [[../../operating-system/operating-system]] — 파일 시스템 / IO 스케줄러
- [[../../distributed-systems/distributed-systems]] — Ceph / HDFS replication
