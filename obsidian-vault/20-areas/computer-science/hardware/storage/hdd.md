---
title: "HDD (Hard Disk Drive)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, hdd, platter, seek, rpm, cmr, smr, hamr, mamr, tler]
---

# HDD (Hard Disk Drive)

**[[storage|↑ 저장 장치]]**

> SSD 가 주류가 된 지금도 cold storage / 대용량 NAS / 백업의 표준. TB 당 가격이 SSD 의 1/5 ~ 1/10.

## 1. 물리 구조

```
  ┌──────────────────────────┐
  │  Top cover               │
  │   ┌──────────┐           │
  │   │ platter 1 │ ◄── 회전 (RPM)
  │   │ ─────────  │           │
  │   │ platter 2 │           │
  │   │ ─────────  │           │
  │   │ platter 3 │           │  → 3-9 platters
  │   └──────────┘           │
  │       ▲                  │
  │       │ head (per surface)│
  │  ─────┴──────  ◄── 액추에이터 암 (seek)
  │       │                  │
  │   Voice Coil Motor (VCM) │
  │                          │
  │  Spindle motor (회전)    │
  └──────────────────────────┘
```

### 주요 구성

| 부품 | 역할 |
| --- | --- |
| **Platter** | 자성 코팅된 알루미늄 / 유리 원판. 1~9 장. 양 면 사용. |
| **Head** | platter 양 면에 1 개씩. 비행 높이 수 nm (사람 머리카락 두께의 1/10,000). |
| **Actuator arm** | head 위치 이동 (radial). |
| **VCM (Voice Coil Motor)** | actuator arm 구동. magnet + coil. |
| **Spindle motor** | platter 회전 (5400/7200/10K/15K RPM). |
| **PCB controller** | host 인터페이스 + servo control + caching. |

### 헤드의 마법
- 비행 높이 ~3 nm (현 세대 HDD).
- 비행 매체 = air bearing (디스크 회전으로 형성).
- 정전 / 진동 / 충격 = head crash → 매체 손상.

## 2. Read/Write 메커니즘

### Write
- head 의 micro-coil 이 자기장 발생.
- platter 의 자성 매체 (보통 CoCrPt) 의 N/S 방향 결정.
- 1 bit = N→S 또는 S→N 의 transition.

### Read
- platter 의 자기장이 head 의 magnetoresistive (MR) sensor 의 저항 변경.
- TMR (Tunneling Magnetoresistance) head 가 현세대.

### PMR (Perpendicular Magnetic Recording)
- 자기 방향이 platter 면에 **수직**.
- 2006 부터 표준 (이전은 LMR — longitudinal).
- 더 작은 비트 표현 가능.

## 3. 핵심 지표

| 지표 | 의미 | 일반값 |
| --- | --- | --- |
| **RPM** | 회전 속도 | 5400 (저전력), 7200 (일반), 10K (Tier 1 enterprise), 15K (Tier 0, 단종 중) |
| **Seek time** | head 이동 평균 시간 | 5–15 ms |
| **Track-to-track seek** | 인접 track 이동 | 0.5–2 ms |
| **Full stroke seek** | 끝→끝 | 10–25 ms |
| **Rotational latency** | head 아래 sector 도착 평균 (= 0.5 회전) | 7200 → 4.17 ms |
| **Sustained throughput** | 연속 read/write | 100–250 MB/s (현세대) |
| **Random IOPS (4K)** | random 작업 | 100–200 |
| **MTBF** | 평균 무고장 시간 | 1.0–2.5 M hours |
| **Annualized Failure Rate** | 연 실패율 | 0.5–2% (소비자), 0.3-0.8% (enterprise) |

### Random access time 식

```
random access time ≈ seek + rotational latency
```

예: 7200 RPM, 9 ms 평균 seek:
- rotational = 60 / 7200 / 2 = 4.17 ms.
- total = 9 + 4.17 ≈ 13 ms.

→ 1 disk 의 random IOPS = 1 / 0.013 ≈ 77 IOPS.
→ 7200 RPM HDD = 100-150 IOPS 가 천장.

## 4. 인터페이스

### SATA (소비자 / NAS)
- 6 Gbps.
- 데스크탑 / 가정용 NAS.
- single port.

### SAS (서버 / enterprise)
- 12 Gbps / 24 Gbps.
- **듀얼 포트** — 한 디스크에 2 컨트롤러 동시 연결 (HA).
- **더 깊은 큐** (256+).
- **T10 DIF/PI** — 데이터 무결성 metadata.
- enterprise storage 의 표준.

## 5. 폼팩터

| 폼팩터 | 크기 | 용량 | 두께 |
| --- | --- | --- | --- |
| 3.5" | 데스크탑 / NAS / server | 1–28 TB | 26.1 mm |
| 2.5" | 노트북 / server (SFF) | 0.5–5 TB | 7 / 9.5 / 12.5 / 15 mm |
| 1.8" | 옛 미니PC / 일부 모바일 | 단종 | — |

3.5" 가 압도적 — 가격·용량·내구성 모두 우수.

## 6. 기록 방식 진화

### CMR (Conventional Magnetic Recording)
- 일반. 각 track 이 독립.
- 모든 random write OK.
- 7200 RPM enterprise / consumer 의 표준.

### SMR (Shingled Magnetic Recording)
- track 을 **기와처럼 겹쳐** 밀도 ↑.
- write head 가 폭이 넓고 read head 가 좁음 → write 가 인접 track 일부 덮음.
- **읽기 OK**, **random write 매우 느림** (zone 전체 rewrite 필요).
- consumer NAS / 일부 archive drive (Seagate Archive, WD Blue) 에 채택.
- ZFS / RAID 에 절대 비추.

#### 종류
- **DM-SMR (Drive-Managed)** — drive 가 알아서 처리. host 가 모름. → **NAS / RAID 에 fail**.
- **HM-SMR (Host-Managed)** — host 가 SMR 의식 + zone 관리. ZFS / 분산 storage 에 OK.
- **HA-SMR (Host-Aware)** — host 가 zone 알지만 drive 도 알아서 처리.

#### SMR 식별
```bash
sudo hdparm -I /dev/sda | grep -i 'Form Factor\|Trusted\|TRIM'
sudo smartctl -a /dev/sda | grep -i 'Form Factor\|Rotation'
```

### HAMR (Heat-Assisted Magnetic Recording)
- 작은 레이저로 기록 시 매체 가열 (~450 °C, ns 단위).
- 더 작은 자기 그레인 사용 가능 → 밀도 ↑.
- Seagate Mozaic 3+ (2024 30 TB+).
- 신뢰성 / 양산 안정성 도전이 컸으나 2023+ 본격 출하.

### MAMR (Microwave-Assisted Magnetic Recording)
- 작은 microwave 로 자기 그레인 회전 가속.
- Western Digital 의 대안.
- HAMR 보다 단순하지만 밀도 한계.

### ePMR (energy-assisted PMR)
- Western Digital 의 MAMR 진화. 일종의 hybrid.
- 20-22 TB drive.

### 미래 — HDMR / BPMR
- HDMR (Heated-Dot MR) — bit 마다 hot spot.
- BPMR (Bit-Patterned MR) — 매체에 미리 결정된 island. 5+ Tb/in² 가능.

## 7. 자기 그레인과 밀도

- 1 bit = 자기 그레인 ~수십개 (현세대) → ~수개 (HAMR).
- 더 작은 그레인 = 더 적은 magnetic energy = 자기 random walk 위험.
- **superparamagnetic limit** = 그레인 크기 한계.
- HAMR / BPMR 이 이 한계를 우회.

### 데이터 밀도 진화

| 연도 | 밀도 (bit / in²) | 대표 |
| --- | --- | --- |
| 1995 | 1 Gb | first PMR consumer |
| 2010 | 600 Gb | 2 TB drive |
| 2015 | 1 Tb | 6 TB |
| 2020 | 1.4 Tb | 16-18 TB |
| 2024 | 2 Tb (HAMR) | 30 TB+ |
| 2027 (예정) | 3 Tb | 50 TB |

## 8. Cache + NCQ

### On-disk cache
- 64-256 MB DRAM on PCB.
- write coalescing + read prefetch.
- 정전 시 데이터 손실 위험 → enterprise 디스크는 cap 또는 battery 보유.

### NCQ (Native Command Queuing) — SATA
- host 가 한 번에 32 명령 전달.
- drive 가 head 위치 / 회전 위치에 맞춰 재정렬.
- 효과: random IOPS ~20% ↑.

### TCQ — SAS
- 더 깊은 큐 (256+).

## 9. TLER / ERC / CCTL — RAID 친화

- desktop 디스크는 read error 시 long retry (~120 초) 후 포기.
- 그 동안 RAID 컨트롤러가 디스크를 fail 로 오해 → drop → array degraded.

### TLER (Time-Limited Error Recovery) — Western Digital
- 7 초 안에 retry 포기 → RAID 가 다른 디스크로 복구.

### ERC (Error Recovery Control) — Seagate
- 같은 기능.

### CCTL — Hitachi / Toshiba

→ **RAID / NAS 용 디스크는 반드시 TLER 지원 grade**:
- WD Red Plus / Red Pro (NAS).
- Seagate IronWolf / IronWolf Pro (NAS).
- WD Gold / Ultrastar / Seagate Exos (enterprise).

일반 (WD Blue, Seagate Barracuda) = TLER 없음.

## 10. SMART (Self-Monitoring, Analysis and Reporting Technology)

### 핵심 attribute

| ID | Name | 의미 |
| --- | --- | --- |
| 1 | Read Error Rate | low-level read 오류율 |
| 5 | Reallocated Sectors Count | bad sector 재할당 수 |
| 9 | Power-On Hours | 가동 시간 |
| 10 | Spin Retry Count | spin-up 재시도 |
| 187 | Reported Uncorrectable Errors | UE |
| 188 | Command Timeout | command timeout |
| 189 | High Fly Writes | head 비행 높이 비정상 |
| 190 | Airflow Temperature | 풍압 온도 |
| 194 | Temperature | 디스크 온도 |
| 196 | Reallocation Event Count | 재할당 시도 횟수 |
| 197 | Current Pending Sector Count | 보류 sector (불안정) |
| 198 | Offline Uncorrectable | offline scan 에서 UE |
| 199 | UDMA CRC Error Count | 케이블 / 컨트롤러 문제 |

### 위험 신호
- **ID 5 / 196 / 197 / 198** 의 값이 0 이상 + 증가 추세 = 디스크 fail 임박.
- **ID 199** 증가 = SATA 케이블 / 컨트롤러 / 전원 문제.
- **온도 60 °C 이상** = aging 가속.

```bash
sudo smartctl -a /dev/sda
sudo smartctl -t short /dev/sda      # 2 분 short test
sudo smartctl -t long /dev/sda       # 6+ 시간 full surface
sudo smartctl -l selftest /dev/sda
sudo smartctl -A /dev/sda            # attribute table
```

## 11. 데이터센터 활용

### Cold storage / 백업
- TB 당 가격 SSD 의 1/5 ~ 1/10.
- AWS Glacier, BackBlaze B2, Wasabi 의 backbone.
- 24/7 가동 enterprise HDD (Seagate Exos, WD Ultrastar).

### HDFS / Ceph
- 분산 storage 의 backing store.
- replication 으로 가용성, HDD 가 raw 용량 제공.

### CCTV / Surveillance
- 연속 sequential write 최적화.
- WD Purple, Seagate SkyHawk.

### Backblaze 의 통계
- 매 분기 디스크 별 AFR (Annualized Failure Rate) 공개.
- Seagate ST12000 / ST14000 / Toshiba MG07 / HGST 등 비교 가능.
- 일반: enterprise drive 0.5-1% AFR, consumer 1-3%.

## 12. SSHD (Hybrid) — 잠시 있었던 idea
- HDD + small NAND cache.
- 자주 쓰는 데이터를 NAND 에 자동 keep.
- 2010-2015 사이 일부 노트북. 현재는 sole SSD 가 표준.

## 13. 진단

```bash
# 기본 정보
lsblk
sudo hdparm -I /dev/sda | head -30
sudo smartctl -i /dev/sda

# 성능 측정
sudo hdparm -Tt /dev/sda

# fio 벤치
fio --rw=read --bs=1M --iodepth=8 --numjobs=1 --runtime=30 \
    --filename=/dev/sda --direct=1 --time_based

# Random IOPS (HDD = 100-200 만 나옴)
fio --rw=randread --bs=4k --iodepth=4 --numjobs=1 --runtime=30 \
    --filename=/dev/sda --direct=1 --time_based

# 자세한 SMART
sudo smartctl -a /dev/sda

# Badblocks (long, read-only)
sudo badblocks -sv /dev/sda

# IO 통계
iostat -dx 1
```

## 14. HDD 노화 / EOL 신호

1. **SMART ID 5 (Reallocated Sectors) 증가** — fail 임박.
2. **Pending sector (197) 증가** — bad sector 후보.
3. **Spin retry (10) > 1** — motor 약해짐.
4. **Read error rate 비정상** — head 또는 매체 문제.
5. **temperature 폭증** — bearing 마모.
6. **Click sound (death click)** — head 또는 actuator fail.

→ 위 신호 1+ 면 즉시 백업 / 교체.

## 15. 함정

1. **SMR 디스크를 RAID 에** — random write 폭주 시 동결. drive 모델 확인 (DM-SMR 식별).
2. **데스크탑 HDD 를 24/7 NAS 에** — 일반 디스크는 8/5 가동 가정 / vibration 약함. enterprise grade 사용.
3. **TLER 없는 디스크를 RAID 에** — 일시 read 지연이 RAID 에서 fail 로 오해.
4. **물리 충격** — 가동 중 충격은 head crash. 노트북 / 외장 운반 주의.
5. **온도 60+ °C** — bearing / 매체 노화 가속.
6. **다른 batch / vendor 혼용** — 같은 batch 디스크의 동일 시점 fail 위험. 의도적 mix.
7. **SMR 모르고 NAS 구매** — 라벨에 "DM-SMR" 명시 안 됨. spec sheet / Reddit 확인.
8. **vibration 절연 무시** — server chassis 의 진동이 디스크 수명 단축.
9. **SMART 무시** — Pending sector 가 늘면 곧 reallocated 됨. 즉시 backup.
10. **HDD 의 sustained random write 평가** — 100 IOPS 가 천장. DB / VM 호스트는 SSD.

## 16. 관련

- [[storage]]
- [[ssd-and-ftl]]
- [[storage-interfaces]] — SATA / SAS
- [[raid-levels]] — TLER / rebuild URE
- [[../../operating-system/operating-system]] — 파일 시스템 / IO 스케줄러
