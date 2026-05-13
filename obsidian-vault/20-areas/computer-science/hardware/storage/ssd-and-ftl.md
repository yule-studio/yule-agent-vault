---
title: "SSD 와 FTL"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, ssd, nand, ftl, wear-leveling, trim, nvme, gc, slc-cache, znc, dwpd]
---

# SSD 와 FTL (Flash Translation Layer)

**[[storage|↑ 저장 장치]]**

> SSD 의 모든 성능·수명·내구성 특성이 NAND Flash 의 비대칭성 (read ≠ program ≠ erase) 과 FTL 의 알고리즘에서 결정된다.

## 1. NAND Flash 의 물리적 단위

| 단위 | 크기 | 동작 | 비고 |
| --- | --- | --- | --- |
| **cell** | 1 floating gate transistor | 1~4 bit 저장 | SLC/MLC/TLC/QLC/PLC |
| **string** | cell 32~128 개 직렬 | — | 3D NAND 의 vertical stack |
| **page** | 16~32 KB | **read / program 단위** | 한 행 cell 들 |
| **block** | page 256~512 개 | **erase 단위** | 4~32 MB |
| **die** | block 수천 개 | 독립 IO 채널 | 1 칩 안 1-4 die |

### 비대칭 동작
- **Read**: page 단위 (50-200 μs). 셀에 손상 없음.
- **Program (write)**: page 단위 (200-1000 μs). 같은 page 에 다시 program 하려면 먼저 erase.
- **Erase**: **block 단위** (2-10 ms). cell 의 floating gate 를 비움.

→ 사용 중인 page 가 가득한 block 안에 빈 page 가 일부 있어도, 그 자리에 새로 못 씀. block 전체를 erase 해야만.

이 비대칭이 SSD 모든 복잡성의 원인.

## 2. NAND 셀 — Floating Gate Transistor

```
       Control gate (WL)
            │
        ┌───┴───┐
        │ ────── │   ← Floating gate (절연체로 둘러싸인 채 전자 저장)
        ├───────┤
        │       │   ← 채널
        └───────┘
       Source  Drain
```

- Floating gate 에 전자 주입 (Fowler-Nordheim tunneling / hot electron injection) → 1.
- 전자 빼냄 (FN tunneling, 반대 방향) → 0.
- Read = WL 에 reference voltage 인가 → 채널 conduct 여부로 0/1 판정.

### MLC/TLC/QLC — 전압 레벨 다중화

- **SLC**: 0 / 1 두 레벨.
- **MLC**: 00 / 01 / 10 / 11 네 레벨 → 셀당 2 bit.
- **TLC**: 8 레벨 → 셀당 3 bit. 인접 레벨 간 voltage margin 좁음 → ECC ↑.
- **QLC**: 16 레벨 → 셀당 4 bit. 매우 좁은 margin, 큰 ECC + low P/E cycle.

| 종류 | 레벨 | bit/cell | P/E cycles | program 시간 | 적용 |
| --- | --- | --- | --- | --- | --- |
| SLC | 2 | 1 | ~100,000 | 25 μs | enterprise cache, 산업용 |
| MLC | 4 | 2 | ~10,000 | 50 μs | 옛 enterprise SSD |
| TLC | 8 | 3 | ~3,000 | 250 μs | 가장 흔함 (소비자 / mid-tier 엔터프라이즈) |
| QLC | 16 | 4 | ~1,000 | 500 μs | 대용량 / read-heavy / cold storage |
| PLC | 32 | 5 | ~150 | — | 시제품 (Solidigm) |

## 3. 3D NAND — 셀 적층

평면 NAND 한계 (~16 nm) 도달 후, 셀을 **수직 적층**.

- 2013 Samsung V-NAND 첫 양산 (24 layer).
- 2025 = **232 layer+** (Micron / SK hynix / Samsung).
- 2026 예상 = 300+ layer.

```
            (위)
   ─── layer 232 ───
   ───      ...      ───
   ─── layer 1   ───
   ─── (bit line)  ───
```

각 layer = 같은 string 의 한 셀. 같은 column 의 셀들이 한 string 으로 연결. 따라서 3D NAND 의 read 는 string 단위 (페이지 안 모든 비트 동시) 로 처리.

### 장점
- 같은 die 면적에 용량 ↑.
- layer 마다 cell 크기를 크게 유지 가능 → P/E cycle 회복 (planar 와 비교).

### 단점
- 제조 복잡 (수십 단계 etch).
- die yield ↓ → 비용 ↑.

## 4. SSD 컨트롤러 — FTL 의 본체

### 컨트롤러 구성

```
     Host (SATA / PCIe / NVMe)
              │
   ┌──────────┴───────────┐
   │   Host Interface     │  AHCI/NVMe protocol
   ├──────────────────────┤
   │   FTL (CPU + RAM)    │  ★ FTL 알고리즘 실행
   ├──────────────────────┤
   │   ECC engine         │  BCH, LDPC
   ├──────────────────────┤
   │   NAND interface     │  Toggle / ONFI 채널
   └──────┬───────────────┘
          │
    ┌─────┴─────┬─────────┐
    │ NAND ch 0 │ ch 1    │ ...  ch 7 / 16 / 32
    │ die 0-7   │ die 0-7 │
    └───────────┴─────────┘
```

- ARM Cortex 또는 자체 RISC. 6-32 코어.
- DRAM 512 MB ~ 8 GB (HMB 일 경우 host RAM 일부 사용).
- **8~32 채널 NAND parallel** — bandwidth 의 핵심.

### FTL 의 핵심 일

| 작업 | 의미 |
| --- | --- |
| **LBA → PBA mapping** | host 의 logical block address → NAND physical address |
| **Wear leveling** | 모든 셀이 균등하게 닳도록 mapping 회전 |
| **Garbage Collection (GC)** | block 안 valid page 만 다른 block 으로 옮기고 erase |
| **Write Amplification (WAF)** | host write 보다 NAND write 가 항상 더 많음 |
| **Bad Block Management** | 결함 block 격리, spare pool 사용 |
| **TRIM 처리** | host 의 "이 LBA 안 써" 알림 → GC 효율 ↑ |
| **Over-provisioning (OP)** | 사용자 숨겨진 NAND 7~28% 여분 |
| **Read disturb 관리** | 같은 page 읽기 너무 많으면 인접 셀 영향 |
| **Data retention** | 오래 안 읽은 데이터의 charge loss 대비 (read scrub) |

## 5. FTL Mapping 방식

| 방식 | 매핑 단위 | 메모리 | 성능 |
| --- | --- | --- | --- |
| **Page-level** | page 단위 | 큼 (1 GB DRAM / 1 TB NAND) | random write 빠름 |
| **Block-level** | block 단위 | 작음 | random write 매우 느림 (block 전체 rewrite) |
| **Hybrid (log-block, BAST 등)** | 일부 log block + 나머지 block | 중간 | 대부분 사용 |
| **DRAM-less + HMB** | page-level but host RAM 사용 | 칩 비용 ↓ | random 4K 느림 (HMB latency) |

### LBA→PBA 매핑 테이블 크기

- 4 KB page mapping = `1 TB / 4 KB = 256 M entries × 8 byte = 2 GB`.
- 그래서 enterprise SSD 는 1 TB 당 DRAM 1 GB 룰.
- DRAM-less SSD 는 매핑을 NAND 에 보관 + 일부 host RAM (HMB) 캐시.

## 6. Write Amplification (WAF)

```
WAF = NAND write 총량 / host write 총량
```

- 이상적 1.0. 실제 1.2 ~ 3.0.
- 주 원인:
  - **GC overhead** — block erase 위해 valid page 이동.
  - **Read-modify-write** — small write 가 큰 page 채우려고 read+merge.
  - **Wear leveling** — static data 도 옮김.
  - **ECC parity** — 사용자 데이터 + ECC 코드워드.

### WAF 줄이는 방법

1. **OP (Over-provisioning) ↑** — GC 가 더 여유롭게 정리.
2. **TRIM 활성** — host 가 deleted LBA 통보.
3. **Sequential workload** — random 보다 WAF 훨씬 낮음.
4. **SLC cache** — 작은 burst 가 TLC 까지 안 가도 됨.

### 사용자 영향
- WAF = 3.0 이면 host 가 1 TB 쓸 때 NAND 는 3 TB 마모.
- 셀 P/E 한계가 같다면 SSD 수명 = **rated TBW / WAF**.

## 7. Garbage Collection (GC)

### 왜 필요한가

- 사용자가 LBA 100 에 데이터 A 쓰고 → 나중에 같은 LBA 100 에 B 로 update.
- FTL 은 B 를 **다른 NAND page 에 새로** 쓰고, A 의 page 를 **invalid** 로 표시.
- 시간 지나면 block 안 valid + invalid page 혼재.
- erase 위해 block 안 valid page 만 다른 block 으로 옮긴 뒤 erase.

### Background GC vs Foreground GC

- **Background GC**: idle 시간에 미리.
- **Foreground GC**: host write 가 빈 page 못 찾으면 즉시 (write 멈춤 → latency spike).

### GC 알고리즘
- **Greedy**: invalid page 가장 많은 block 부터 GC.
- **Cost-Benefit / Age-based**: invalid 비율 + 마지막 invalidate 시간 고려.
- **Hot/Cold separation**: 자주 update 되는 hot 데이터와 안 변하는 cold 데이터 분리.

## 8. Wear Leveling

- 셀 P/E 한계 (TLC 3000) 때문에 일부 block 만 닳으면 SSD 가 일찍 죽음.
- FTL 이 모든 block 의 erase count 추적 → 가급적 균등하게.

| 종류 | 의미 |
| --- | --- |
| **Dynamic** | 새 write 가 가장 덜 닳은 block 으로 |
| **Static** | static 데이터 (오래 안 변한) 도 가끔 옮겨 다른 block 도 닳게 |

## 9. SLC Caching (TLC/QLC SSD)

- TLC/QLC SSD 의 빈 영역 일부를 **SLC 모드**로 사용 (1 bit/cell 만 저장).
- 빠른 write (SLC 25 μs vs TLC 250 μs).
- 캐시 소진 후 native TLC 속도로 ↓ ("clip", "performance cliff").

### 종류

| 종류 | 의미 |
| --- | --- |
| **Static SLC** | 고정 영역 SLC. 작지만 일정 burst 보장. |
| **Dynamic SLC** | 빈 공간 비율에 따라 가변. 빈 공간 ↑ → SLC 캐시 ↑. |
| **Hybrid** | 두 가지 혼합. |

### 사용자 체감
- 1 GB 영상 복사 → SLC 캐시 ↑, 빠름.
- 50 GB 영상 복사 → 캐시 소진 후 native TLC 속도 (~600 MB/s, 1 GB/s+ 에서 떨어짐).

## 10. TRIM / Discard

- 파일 시스템이 file delete → LBA 의 데이터가 "garbage" 됐다고 SSD 에 알림.
- FTL 이 그 LBA 의 PBA 를 invalid 로 표시 → GC 가 빨리 정리.
- TRIM 없으면 SSD 는 deleted LBA 도 valid 로 간주 → WAF ↑, 수명 ↓.

```bash
# Linux
sudo fstrim -av                       # 수동 TRIM
sudo systemctl status fstrim.timer    # 주기적 자동 (기본 weekly)

# Mount option
/dev/nvme0n1p1 / ext4 defaults,discard 0 1  # 옛 방식 (per-delete TRIM, 느릴 수 있음)
```

## 11. Over-provisioning (OP)

- 사용자 모르게 SSD 안에 숨겨진 spare NAND.
- 컨슈머: 7% (1024 GB → 사용자 1000 GB).
- 엔터프라이즈: 14~28% (1920 GB → 사용자 1600 GB).

### OP 효과
- GC 더 효율적 (옮길 page 가 더 적은 빈 곳).
- WAF ↓ → 수명 ↑.
- sustained write 성능 ↑.

### 사용자 OP
- SSD 의 일부를 partition 안 하거나 unpartitioned 로 두면 자동 OP 처럼 작동.
- "secure erase 후 80% 만 사용" 처럼.

## 12. Read Disturb / Data Retention

### Read Disturb
- 같은 page 를 너무 자주 read → WL 의 미세한 전압이 인접 cell 의 floating gate 에 영향.
- 일정 read 횟수 후 ECC error rate ↑ → FTL 이 자동으로 다른 곳에 copy.

### Data Retention
- 안 읽고 안 쓴 채 오래 두면 charge loss → 1-5 년 후 데이터 손실 가능.
- 컨트롤러가 주기적으로 **background read scrub** + 위험 page 는 다른 곳에 rewrite.
- 비활성 SSD (장기 보관) 는 1 년에 한 번 전원 인가 권장.

## 13. NVMe — 큐 구조 + 명령

### 큐
- **64K queue × 64K depth** (이론).
- 실제 NVMe SSD = 8~128 큐.
- 멀티 코어 IO 가 코어 당 1 큐 → 락 없이 분산.

### Admin Queue
- Identify, Create/Delete I/O Queue, Get Log, Format, Firmware Download.

### IO Queue
- Read, Write, Flush, Compare, Write Zeroes, Dataset Management (TRIM), Reservation.

### Namespace
- 한 SSD 안 여러 namespace = 가상 분리. 다른 namespace 끼리 OP / 보안 / 액세스 정책 다르게.

## 14. ZNS (Zoned Namespace)

NAND 의 block 구조를 host 에 그대로 노출.

- host = log-structured 파일 시스템 (RocksDB, F2FS, ZFS) 이 직접 zone 관리.
- FTL 의 GC / WAF 대부분 사라짐.
- host 가 잘 못 쓰면 (random write to multiple zone) 성능 폭락.

### 적용 예
- Western Digital Ultrastar DC ZN540.
- RocksDB / TerarkDB / Aerospike 의 ZNS-aware build.

## 15. NVMe-oF (Over Fabrics)

NVMe 명령을 네트워크 위로.

| 전송 | 비고 |
| --- | --- |
| NVMe/TCP | 가장 보편. 100 GbE+ |
| NVMe/RDMA (RoCE v2 / iWARP) | 낮은 latency |
| NVMe/FC | Fibre Channel SAN |

→ composable infrastructure / data lake / AI 학습 dataset 공유.

## 16. DWPD (Drive Writes Per Day)

엔터프라이즈 SSD 의 보증 spec.

```
TBW (Total Bytes Written) = DWPD × 용량(TB) × 보증기간(일)
```

예: DWPD 3, 1.92 TB, 5 년 = `3 × 1.92 × 365 × 5 = 10,512 TB ≈ 10 PB`.

| 등급 | DWPD | 용도 |
| --- | --- | --- |
| Read-intensive | 0.3 ~ 1 | OS, 일반 data |
| Mixed-use | 1 ~ 3 | DB, VM, 가상화 |
| Write-intensive | 3 ~ 10+ | DB WAL, 로그, 캐시 |

DB WAL, Kafka, Redis snapshot 같은 write 폭증 워크로드는 **PLP (Power Loss Protection) + DWPD ≥ 3** 필수.

## 17. PLP (Power Loss Protection)

- 정전 시 in-flight write 보호.
- 컨트롤러 위 캐패시터 (큰 tantalum cap 또는 super-cap).
- 전원 잃으면 DRAM cache 의 데이터를 NAND 로 flush.
- 엔터프라이즈 SSD 의 distinguishing 기능 — 컨슈머 SSD 는 거의 없음.

## 18. NVMe SSD 폼팩터

| 폼팩터 | 인터페이스 | 전력 | 용도 |
| --- | --- | --- | --- |
| **2.5" SATA** | SATA | 5-7 W | 노트북 / 데스크탑 |
| **M.2 2280** | NVMe (PCIe x2/x4) 또는 SATA | 8.25 W | 노트북 / 데스크탑 |
| **U.2 / U.3** | NVMe PCIe x4 | 25 W | 서버 2.5" hot-swap |
| **AIC (Add-in Card)** | NVMe PCIe x4/x8 | — | 워크스테이션 / 서버 |
| **E1.S** (5.9~25 mm) | NVMe PCIe x4 | 12-25 W | 데이터센터 thin ruler |
| **E1.L** | NVMe PCIe x4 | 25 W | 긴 ruler, 대용량 |
| **E3.S / E3.L** | NVMe / CXL | 25-40 W | 새 EDSFF 표준 |

## 19. 진단 / 측정

```bash
# 기본 정보
sudo nvme list
sudo nvme id-ctrl /dev/nvme0n1 | head -40
sudo nvme id-ns /dev/nvme0n1 -n 1
sudo nvme smart-log /dev/nvme0n1
sudo nvme fw-log /dev/nvme0n1

# 핵심 SMART 필드
#   Percentage Used                — 수명 사용률 (100% = 보증 끝)
#   Available Spare                — 예비 NAND 잔여 %
#   Media and Data Integrity Errors — ECC 실패 누적
#   Temperature                    — 운영 온도 (70 °C 넘으면 throttling)
#   Power On Hours
#   Data Units Read/Written        — host I/O 통계 (× 512 KB)

# 자세한 SMART (consumer)
sudo smartctl -a /dev/nvme0n1

# 성능 측정
fio --rw=randread --bs=4k --iodepth=32 --numjobs=4 --runtime=30 \
    --filename=/dev/nvme0n1 --direct=1 --time_based --group_reporting

fio --rw=randwrite --bs=4k --iodepth=32 --numjobs=4 --runtime=30 \
    --filename=/dev/nvme0n1 --direct=1 --time_based --group_reporting

# Sustained write — 캐시 소진 후 native 속도 확인
fio --rw=write --bs=128k --iodepth=4 --numjobs=1 --runtime=600 \
    --filename=/dev/nvme0n1 --direct=1 --time_based

# TRIM 상태
sudo fstrim -v /
```

## 20. 함정

1. **TLC/QLC 의 sustained write 폭락** — burst 캐시 소진 후 native 속도로 떨어짐. `clip` 후 ~1/3 속도.
2. **GC 와 사용자 IO 경쟁** — 80% 이상 채우면 GC 더 빈번 → write 성능 ↓. 20%+ 빈 공간 권장.
3. **DRAM-less SSD 의 random performance** — HMB 사용 — random 4K 가 매우 느림.
4. **PLP 없는 SSD 를 DB 서버에** — 정전 시 in-flight write 손실. DB 가 corruption.
5. **TBW 한계 무시** — TLC TBW ~ 0.6 × 용량 × P/E_max.
6. **bit rot** — read 안 하고 오래 둔 SSD 의 데이터 손실. 1-5 년에 한 번 read scrub.
7. **SMR HDD 와 혼동** — SMR 은 zone 비슷한 제약. SSD ZNS 와는 다른 layer.
8. **TRIM 비활성 채 LVM / mdraid** — 일부 stacked layer 에서 TRIM 전달 안 됨. `dmsetup` discard 지원 확인.
9. **fstrim 너무 자주 / 너무 드물게** — 너무 자주 = TRIM 자체 부담. 너무 드물게 = WAF ↑. 기본 weekly 가 적절.
10. **firmware 업데이트 무시** — 컨슈머 SSD 도 큰 firmware bug 수정 흔함 (Samsung 990 Pro 의 수명 카운터 버그 등).
11. **온도 80 °C 초과 throttling** — Gen5 SSD heatsink 필수.
12. **secure erase 명령 혼동** — `nvme format` vs `nvme sanitize`. sanitize 가 crypto erase / block erase 의 표준.

## 21. 관련

- [[storage]]
- [[hdd]]
- [[storage-interfaces]] — SATA / NVMe / M.2 / U.2 / EDSFF
- [[raid-levels]] — SSD RAID 의 함정 (TLER 다름)
- [[../bus-io/pcie]] — NVMe 가 올라타는 layer
- [[../../operating-system/operating-system]] — IO scheduler, file system, fstrim
