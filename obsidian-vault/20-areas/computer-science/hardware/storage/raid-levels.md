---
title: "RAID 레벨"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, raid, parity, zfs, mdadm, ure, lvm, ceph, dRAID]
---

# RAID 레벨

**[[storage|↑ 저장 장치]]**

> 디스크 여러 개를 조합해 가용성·성능·용량 trade-off 를 잡는 표준.

## 1. RAID 레벨 한눈에

| 레벨 | 디스크 수 | 손실 허용 | 가용 용량 | random read | random write | 비고 |
| --- | --- | --- | --- | --- | --- | --- |
| **RAID 0** | ≥2 | 0 | 100% | ↑↑ | ↑↑ | 가용성 0. 캐시 / 임시. |
| **RAID 1** | 2 (또는 N) | N-1 | 1/N | ↑ | 단일 | mirror. |
| **RAID 5** | ≥3 | 1 | (N-1)/N | ↑ | ↓ (parity) | 옛 표준. 큰 HDD 환경 비추. |
| **RAID 6** | ≥4 | 2 | (N-2)/N | ↑ | ↓↓ | 대용량 HDD 표준. |
| **RAID 10** | ≥4 (2N) | 운 좋으면 N/2 | 50% | ↑↑ | ↑ | mirror + stripe. |
| **RAID-Z2 / Z3 (ZFS)** | ≥4 / 5 | 2 / 3 | ≈(N-2 or 3)/N | ↑ | 보통 | ZFS dynamic stripe. |
| **dRAID (ZFS)** | ≥10+ | configurable | configurable | ↑ | ↑ | 빠른 rebuild. |
| **erasure coding (Ceph / S3)** | flex | configurable | 매우 효율 | ↑ | ↓ | 분산 storage. |

## 2. RAID 0 — Striping

### 동작
- N 디스크에 데이터 stripe (일정 chunk 크기씩 분산).
- 모든 IO 가 N 디스크 동시 → throughput × N.

### 문제
- **중복성 0**. 1 디스크 fail = 전체 손실.
- N 디스크의 한 디스크 fail 확률 = 1 - (1-p)^N (대략 N × p).

### 사용처
- 임시 캐시 (Spark shuffle, FFmpeg scratch).
- 비디오 편집 scratch disk.
- 절대 production data 에는 안 됨.

## 3. RAID 1 — Mirroring

### 동작
- 같은 데이터를 2 (또는 N) 디스크에 복제.
- read 는 양쪽 분산 가능 → throughput ↑.
- write 는 양쪽 동시.

### 장점
- 1 디스크 fail 시 무중단.
- read 성능 ↑.

### 단점
- 가용 용량 50% (2-disk).
- write 가 단일 디스크 수준.

### 사용처
- OS boot drive (2-disk mirror).
- 소형 NAS 의 single-volume.

## 4. RAID 5 — Distributed Parity

### 동작

```
Disk 1:  A1  B1  C1  Pd
Disk 2:  A2  B2  Pc  D1
Disk 3:  A3  Pb  C3  D2
Disk 4:  Pa  B3  C2  D3
```

- N 디스크에 데이터 + 1 디스크 분량 parity 분산.
- parity = XOR of data blocks: `P = D1 ⊕ D2 ⊕ D3 ⊕ ...`
- 1 디스크 fail 시 다른 디스크의 P + remaining data 로 복구.

### Write 의 비용

**RMW (Read-Modify-Write)**:
1. 영향 받는 stripe 의 다른 데이터 + parity read.
2. 새 데이터로 parity 재계산.
3. 새 데이터 + 새 parity write.

→ 1 write = 2 read + 2 write = **write 4 배 amplification**.

### URE Rebuild 문제 — RAID 5 가 큰 HDD 에 비추인 이유

**URE (Unrecoverable Read Error)** = HDD 사양상 평균 `1 in 10^14 bit` (consumer) ~ `1 in 10^16 bit` (enterprise) 빈도로 read 실패.

10 TB drive 1 개 = 80 × 10^12 bit. consumer 사양이면 평균 1 회 / 디스크 read 1 회 풀스캔.

→ **8 × 10 TB RAID 5 의 rebuild**:
- 70 TB read 필요.
- URE 누적 확률 ≈ 1 - (1 - 1/10^14)^(70 × 8 × 10^12) ≈ 50%.
- rebuild 중 두 번째 디스크 fail → **전체 데이터 손실**.

→ enterprise drive (10^16 URE) 라도 rebuild 시간 길어 위험.

### 권장 대안
- **RAID 6** (2 디스크 fail 허용).
- **RAID 10** (rebuild 가 한 mirror pair 만).
- **ZFS RAID-Z2 / Z3** + 정기 scrub.

## 5. RAID 6 — Dual Parity

### 동작

```
Disk 1:  A1  B1  C1  Pd  Qd
Disk 2:  A2  B2  Pc  Qc  D1
Disk 3:  A3  Pb  Qb  C2  D2
Disk 4:  Pa  Qa  B3  C3  D3
Disk 5:  Qa' A4  B4  C4  D4   (Q parity)
```

- parity P (XOR) + parity Q (Reed-Solomon GF(2^8) 다항식).
- 어떤 2 디스크 fail 도 복구.

### 비용
- Write 가 RAID 5 보다 더 ↓ (parity 2 개 갱신).
- random small write = 6 IO (3 read + 3 write).

### 사용처
- 큰 HDD (≥10 TB) 의 현실적 default.
- 5+ 디스크 NAS.

## 6. RAID 10 — Stripe of Mirrors

### 동작

```
Stripe across mirrors:

  [Disk 1 ↔ Disk 2]  (mirror)
  [Disk 3 ↔ Disk 4]  (mirror)
  [Disk 5 ↔ Disk 6]  (mirror)
        ↓
  stripe across the mirrors
```

- 먼저 mirror, 그 다음 stripe.
- 가용 용량 50%.

### 장점
- 빠른 read + 빠른 write.
- mirror pair 의 한 쪽씩만 죽으면 N/2 디스크 fail 가능.
- rebuild = mirror pair 의 한 쪽 copy = 빠름.

### RAID 0+1 vs RAID 1+0
- **RAID 0+1**: stripe 먼저, 그 stripe 를 mirror.
- **RAID 1+0**: mirror 먼저, 그 mirror 들을 stripe.
- **RAID 10 = 1+0**.
- 1+0 가 더 안정 (한 디스크 fail 이 1 mirror pair 만 영향).

## 7. ZFS RAID-Z / dRAID

### RAID-Z1 / Z2 / Z3
- Z1 = parity 1 (RAID 5 비슷). Z2 = parity 2 (RAID 6 비슷). Z3 = parity 3.
- 차이: **dynamic stripe size** — partial stripe 처리. RAID 5 의 RMW 문제 완화.
- **scrub** — 주기적 모든 데이터 read + checksum 검증 + 손상 시 parity 로 자동 복구.
- 끝-끝 checksum 으로 silent corruption 도 잡음.

### dRAID (ZFS 2.1+)
- distributed parity. spare disk 공간이 array 안에 분산.
- rebuild 시 모든 디스크가 동시 write 참여 → **rebuild 가 매우 빠름**.
- 큰 NAS / data lake (100+ 디스크) 에서 RAID-Z 보다 우수.

```
ZFS pool
├── vdev 1 (dRAID2:8d:1s)  ← 8 data + 2 parity + 1 distributed spare
├── vdev 2 (dRAID2:8d:1s)
└── vdev 3 (mirror)
```

### ZFS pool 구조
- pool = 1+ vdev.
- vdev = single / mirror / RAID-Z / dRAID.
- pool 의 가용성 = 가장 약한 vdev.

## 8. Software RAID (Linux MD / mdadm)

```bash
# RAID 5 생성
sudo mdadm --create /dev/md0 --level=5 --raid-devices=4 \
  /dev/sd[abcd]

# 확인
cat /proc/mdstat
sudo mdadm --detail /dev/md0
sudo mdadm --examine /dev/sda

# spare 디스크 추가
sudo mdadm --add /dev/md0 /dev/sde

# fail 시뮬레이션
sudo mdadm --fail /dev/md0 /dev/sda
sudo mdadm --remove /dev/md0 /dev/sda
sudo mdadm --add /dev/md0 /dev/sda    # 새 디스크 추가

# resync 진행
watch -n 2 'cat /proc/mdstat'
```

## 9. Hardware RAID vs Software RAID

| 비교 | 하드웨어 RAID | 소프트웨어 RAID |
| --- | --- | --- |
| **컨트롤러** | 별도 ASIC + DRAM + BBU/FBWC | OS 가 처리 (mdadm / LVM / ZFS / btrfs) |
| **OS 부담** | 적음 | 일부 CPU 사용 (현 멀티 코어는 무시) |
| **BBU / FBWC** | write cache 안정 보장 | ZFS sync write 자체 안정 |
| **벤더 고정** | 같은 model 컨트롤러 필요 | 어떤 호스트로도 옮김 |
| **유연성** | RAID 5/6/10 정도 | 거의 무한 (Z3, dRAID, hybrid 등) |
| **monitoring** | vendor 도구 | OS 표준 (mdstat, zpool) |
| **rebuild 속도** | 컨트롤러 의존 | 모든 OS BW 사용 |

현대 데이터센터는 보통 **HBA + ZFS / Ceph / RAID-Z** 로 소프트웨어 RAID 채택. 하드웨어 RAID 는 옛 enterprise / 일부 storage array 에 잔존.

## 10. Erasure Coding — 분산 storage

RAID 의 분산 시스템 버전.

### Reed-Solomon EC
- N data shards + M parity shards.
- 임의 M shard fail 까지 복구.
- Hadoop HDFS, Ceph, MinIO, AWS S3 의 backbone.

### 효율 비교
- RAID 5 = N+1 (1 fail 허용, 1/N overhead).
- RAID 6 = N+2 (2 fail, 2/N).
- EC 8+3 = 8 data + 3 parity (3 fail, 3/11 ≈ 27% overhead).

### EC vs Replication
- Replication 3x = 200% overhead, 2 fail 허용.
- EC 8+3 = 37.5% overhead, 3 fail 허용.
- → 큰 cluster 에서 EC 가 훨씬 효율적.

### Latency 차이
- Replication: 빠른 read (any replica 골라).
- EC: read 시 N shard 필요 → 약간 느림.

## 11. RAID 의 한계 — 백업 아님

**RAID 는 가용성 (uptime) 을 위한 것이지 데이터 보호가 아니다.**

RAID 가 못 막는 것:
- 파일 실수 삭제.
- ransomware 가 모든 파일 암호화.
- `rm -rf /`.
- 화재 / 침수 / 도난.
- 소프트웨어 버그 (DB corruption).

→ **3-2-1 백업 규칙**:
- 3 copy of data.
- 2 different media types.
- 1 offsite.

## 12. RAID 의 운영 함정 — 자세히

### TLER / ERC / CCTL — 의 차이
- **TLER (Time-Limited Error Recovery)**: Western Digital.
- **ERC (Error Recovery Control)**: Seagate.
- **CCTL (Command Completion Time Limit)**: Samsung / Hitachi.

같은 기능 다른 이름:
- 디스크가 read error 시 long retry (~2 분) 대신 7 초 안에 포기.
- RAID 컨트롤러가 short retry 후 즉시 다른 디스크에서 복구.

**일반 (desktop) 디스크 = TLER 없음**:
- 일시 read 지연을 RAID 가 fail 로 오해 → 디스크 drop → array degraded.
- WD Red Plus / Seagate IronWolf / Toshiba N-series 같은 NAS-grade 디스크 권장.

### Rebuild 동안 사용자 IO
- rebuild 중 read 부하 ↑ + URE 누적 위험 ↑.
- **rebuild 우선순위 낮춤** (mdadm 의 sync_speed_max 설정).
- 또는 야간 rebuild.

### Same-batch 디스크 동일 시점 fail
- 같은 batch (생산일 / 공장) 디스크가 동일 운영시간에 fail 하는 현상.
- 대응: 디스크 staggered 교체, 다른 vendor 의 디스크 mix.

### 온도 / vibration
- enterprise NAS / server 의 vibration 이 디스크 수명 단축.
- vibration 절연 (rubber grommet) + 적절한 chassis.

## 13. Practical 시나리오 — 추천

### 가정 NAS (4 bay, 12 TB drive)
- **추천**: RAID 6 (2 parity) 또는 ZFS RAID-Z2.
- 12 TB × 4 = 48 TB raw, 가용 24 TB.
- 2 디스크 fail 까지 안전.

### 8-bay NAS (TrueNAS / 가정 서버)
- **추천**: ZFS RAID-Z2 또는 dual mirror (4 × 2).
- 6 disk RAID-Z2 = 4 디스크 분량 가용.
- scrub 매주 자동.

### 대형 데이터 lake (50+ disk)
- **추천**: dRAID2:8d:2s (8 data + 2 parity + 2 spare).
- rebuild 가 distributed → 시간 ↓.
- 또는 Ceph + EC 8+3.

### SSD-only DB 서버
- **추천**: RAID 10 (4 NVMe drive).
- random write IOPS 우선. parity 부담 없음.
- 또는 SW raid 안 쓰고 application-level replication.

### SSD-only Object storage
- **추천**: Ceph + EC.
- 노드 별 single SSD or RAID 10 underlying.

## 14. 진단 / 모니터링

```bash
# mdadm (Linux SW)
cat /proc/mdstat
sudo mdadm --detail /dev/md0
sudo mdadm --examine /dev/sd[a-d]

# 자동 monitoring
sudo mdadm --monitor --daemonise /dev/md0 --mail=ops@example.com

# ZFS
zpool status -v
zpool list
zpool history
zpool scrub tank                # 시작
zpool iostat -v 1               # 디스크 별 IO
zpool get all tank

# HW RAID (Broadcom MegaRAID / LSI)
storcli /c0 show
storcli /c0/v0 show
storcli /c0/e252/s0 show all
sudo MegaCli -PDList -aAll
sudo MegaCli -LDInfo -Lall -aAll

# 디스크 자체 SMART
sudo smartctl -a /dev/sda
sudo smartctl -t long /dev/sda      # 6+ 시간 self-test

# IO 통계
iostat -dx 1
sudo apt install sysstat
```

## 15. ZFS scrub — silent corruption 방어의 핵심

- 주기적 모든 데이터 + parity read.
- 각 block 의 checksum 검증.
- 불일치 발견 시 다른 mirror / parity 로 자동 정정.
- 일반 RAID 가 잡지 못하는 silent corruption (cosmic ray / bit rot / 컨트롤러 bug) 방어.

```bash
zpool scrub tank
zpool status                # scrub 진행률
# scan: scrub in progress since Mon May 13 ...
#       150G scanned, 50G repaired
```

권장 주기: 월 1회 (mirror), 분기 1회 (RAID-Z).

## 16. 함정

1. **RAID = 백업 아님** — 파일 삭제 / ransomware / 휴먼 에러는 RAID 가 막지 못함.
2. **RAID 5 + 큰 HDD** — rebuild URE 누적 → 두 번째 fail. RAID 6 / Z2 권장.
3. **rebuild 도중 사용자 IO 풀가동** — 더 길어지고 fail 확률 ↑. priority 낮춤.
4. **다른 제조사 / 펌웨어 디스크 혼용 in RAID** — 일부 펌웨어 차이가 stripe 의 latency variance 키움.
5. **TLER/ERC 없는 디스크를 RAID 에** — desktop 디스크의 long retry 가 RAID 에서 fail 로 오해.
6. **온도 / vibration 무시** — 같은 batch 디스크 동일 시점 fail.
7. **HW RAID 컨트롤러 BBU 만료** — BBU 가 죽으면 write cache off → 성능 1/10.
8. **ZFS pool 의 vdev 추가 후 reverse 불가** — vdev 한 번 추가하면 제거 못 함 (ZFS 2.x 의 일부 vdev removal 외).
9. **single point of HW RAID** — RAID 컨트롤러 자체가 SPOF. 같은 model 컨트롤러로 교체 가능해야 함.
10. **EC 의 read 부담** — 작은 random read 가 N shard 동시 fetch 필요 → bandwidth 사용량 ↑.
11. **mdadm 의 bitmap intent log 부재** — 옛 default. 디스크 fail 후 full resync 부담. 현 default 는 internal bitmap.
12. **ZFS 의 dedup 기본 활성화** — 메모리 5 GB / 1 TB data 필요. 잘 모르면 OFF.

## 17. 관련

- [[storage]]
- [[hdd]]
- [[ssd-and-ftl]]
- [[storage-interfaces]]
- [[../../operating-system/operating-system]] — 파일 시스템, IO 스케줄러
- [[../../distributed-systems/distributed-systems]] — Ceph, HDFS, EC, replication
