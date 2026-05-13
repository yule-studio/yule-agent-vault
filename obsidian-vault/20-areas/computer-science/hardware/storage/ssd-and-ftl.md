---
title: "SSD 와 FTL"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, ssd, nand, ftl, wear-leveling, trim, nvme]
---

# SSD 와 FTL (Flash Translation Layer)

**[[storage|↑ 저장 장치]]**

## 1. NAND Flash 의 물리

- **page** — read/program 최소 단위. 4–16 KB.
- **block** — erase 최소 단위. 수십 MB.
- **page 는 1 회 program 후 erase 전에 다시 program 불가** (적어도 같은 페이지 안에서).
- 따라서 **write before erase 필요**, 그것도 page 가 아니라 block 단위.

```
   Block (24 MB)
   ┌─────────────┐
   │  Page 0     │ 16 KB
   ├─────────────┤
   │  Page 1     │
   ├─────────────┤
   │   ...       │
   ├─────────────┤
   │  Page N-1   │
   └─────────────┘
```

이 비대칭성이 SSD 모든 복잡성의 원인.

## 2. NAND 타입

| 종류 | bit/cell | P/E cycles | program 시간 | 가격 |
| --- | --- | --- | --- | --- |
| SLC | 1 | 100,000 | ~25 μs | 비싸다 |
| MLC | 2 | 10,000 | ~50 μs | 중 |
| TLC | 3 | 3,000 | ~250 μs | 싸다 |
| QLC | 4 | 1,000 | ~500 μs | 가장 싸다 |
| PLC | 5 | ~150 | (시제품) | — |

**3D NAND**: 셀을 수직 적층 (232 layer+, 2025). 면적 효율 ↑.

### SLC caching
- TLC/QLC SSD 가 빈 영역의 일부를 SLC 모드로 사용 → 짧은 burst write 에 빠른 속도. 캐시 소진되면 native TLC/QLC 속도로 떨어짐 ("clip").

## 3. FTL (Flash Translation Layer)

SSD 컨트롤러가 하는 일:

1. **Logical → Physical Mapping** — LBA → PBA (Page) 매핑 테이블 (~1/1000 of capacity in DRAM cache).
2. **Wear leveling** — 모든 셀이 균등하게 닳도록 매핑을 회전.
   - **Dynamic** — write 빈도 균등화.
   - **Static** — 데이터 자체를 옮겨 사용 안 되는 셀도 닳게.
3. **Garbage Collection (GC)** — block 안의 살아남은 page 를 다른 block 으로 옮기고 erase. GC 가 host write 와 경쟁하면 성능 ↓ (sustained write 가 burst 보다 훨씬 느림).
4. **Write Amplification (WA)** — host 가 1 GB 쓰는데 NAND 에는 1.2~3.0 GB 쓰임. WA = NAND write / host write.
5. **Bad Block Management** — 결함 block 격리.
6. **TRIM 처리** — OS 가 deleted LBA 통보. GC 가 빈 page 로 처리.
7. **Over-provisioning (OP)** — 사용자 모르게 7~28% 추가 NAND. GC 효율 / 수명 ↑.
8. **PLP (Power Loss Protection)** — 내장 캐패시터로 정전 시 DRAM cache 를 NAND 에 flush. enterprise 필수.

## 4. NVMe 큐 / 폼팩터

### 큐 구조
- **64K queues × 64K depth** (이론). 실 NVMe SSD = 8~128 큐.
- 큐 분산이 멀티 코어 IO 의 핵심 (코어 당 1 큐 → 락 없음).

### 폼팩터

| 폼팩터 | 인터페이스 | 용도 |
| --- | --- | --- |
| **2.5" SATA** | SATA 6 Gbps | 노트북 / 데스크탑 |
| **M.2** | NVMe (PCIe x4) 또는 SATA | 노트북 / 데스크탑 |
| **U.2 / U.3** | NVMe PCIe x4 | 서버 2.5" hot-swap |
| **AIC (Add-in Card)** | NVMe PCIe x4 또는 x8 | 워크스테이션 / 서버 |
| **E1.S / E1.L** | NVMe | 데이터센터 ruler, hot-swap |
| **E3.S / E3.L** | NVMe / CXL | 신 데이터센터 표준 |

### 속도

- SATA SSD: 500–550 MB/s
- NVMe PCIe 3.0 x4: 3.5 GB/s
- NVMe PCIe 4.0 x4: 7 GB/s
- NVMe PCIe 5.0 x4: 14 GB/s
- NVMe PCIe 6.0 x4 (예정): 28 GB/s

random read latency: 50–200 μs (HDD 대비 100×).

## 5. SSD 등급 (DWPD)

**DWPD (Drive Writes Per Day)** — 보증 기간 동안 매일 디스크 전체 용량의 몇 배를 써도 보장되는가.

| 등급 | DWPD | 용도 |
| --- | --- | --- |
| Read-intensive | 0.3–1 | OS / 일반 데이터 |
| Mixed-use | 1–3 | DB / VM |
| Write-intensive | 3–10+ | DB WAL / 캐시 / metadata |

DB WAL / Kafka / Redis snapshot 같은 write 폭증 워크로드는 **enterprise SSD with PLP + DWPD ≥ 3** 필수.

## 6. SSD 모니터링

```bash
sudo smartctl -a /dev/nvme0n1
# 핵심 필드:
#   Percentage Used                — 수명 사용률 (100% = 보증 끝)
#   Media and Data Integrity Errors — 비트 오류
#   Temperature                    — 운영 온도 (70 °C 넘으면 throttling)
#   Power On Hours
#   Data Units Read/Written
#   Available Spare                — 예비 NAND 잔여

sudo nvme smart-log /dev/nvme0n1
sudo nvme list
sudo nvme id-ctrl /dev/nvme0n1
fio --rw=randread --bs=4k --iodepth=32 --numjobs=4 --runtime=30 ...
```

## 7. ZNS (Zoned Namespace) / OCSSD

NAND 의 block 구조를 OS 에 그대로 노출 → FTL 의 GC / WA 를 호스트가 직접 관리. 데이터센터 (RocksDB / ZFS) 에 일부 채택.

## 8. 함정

1. **TLC/QLC 의 sustained write 폭락** — burst 캐시 소진 후 native 속도로 떨어짐. 큰 파일 복사 시 체감.
2. **GC 와 사용자 IO 경쟁** — 항상 80% 이상 채워두면 GC 가 더 빈번 → write performance 저하. 20%+ 빈 공간 권장.
3. **DRAM-less SSD 의 random performance** — 매핑 테이블이 호스트 RAM (HMB) 또는 NAND 에 있어 random 4K 가 매우 느림.
4. **PLP 없는 SSD 를 DB 서버에** — 정전 시 in-flight write 손실.
5. **TBW 한계 무시** — TLC 의 TBW ~ 0.6 × 용량 × P/E. 600 GB drive 의 TBW ~ 1080 TB.
6. **bit rot** — read 안 하고 오래 두면 캐패시터 누설로 데이터 손실. 1–5 년에 한 번 read scrub 권장.

## 9. 관련

- [[storage]]
- [[hdd]]
- [[storage-interfaces]]
- [[../bus-io/pcie]] — NVMe 가 올라타는 layer
