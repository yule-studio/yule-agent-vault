---
title: "DRAM 셀 동작"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, dram, refresh, sense-amp, row, column, bank, hbm, cxl, trcd, trp, cl]
---

# DRAM 셀 동작

**[[memory|↑ 메모리]]**

> DRAM 의 모든 latency / bandwidth / power 특성이 결국 이 1T1C 셀의 물리에서 시작한다.

## 1. 1-T 1-C 셀 — 가장 작은 단위

```
   word line (WL)
       │
      ─┴─    ← access transistor (n-MOS)
       │
       │ ── bit line (BL)
       │
      ───    ← storage capacitor (~20 fF)
       │
      GND (또는 Vdd/2 plate)
```

### 동작 원리
- 캐패시터에 전하 있으면 **1**, 없으면 **0**.
- access transistor 가 캐패시터와 bit line 을 잇는 스위치.
- WL 가 HIGH → transistor ON → 캐패시터와 BL 가 연결.

### 왜 "Dynamic" 인가
- 캐패시터의 누설로 ~ms 단위에서 전하가 빠짐.
- **refresh** 가 필요. 표준 64 ms 안에 모든 row 한 번씩 다시 charge.
- 대조: **SRAM** 은 6-T cross-coupled inverter 라 자체 안정 (static).

### Read 가 destructive 인 이유

- 캐패시터의 전하 (~20 fF × 1 V = 20 fC) 가 bit line (~수백 fF) 에 분배.
- BL 의 전압 변화는 매우 작음 (~50-100 mV).
- **Sense Amp** 가 이 작은 변화를 Vdd 또는 GND 로 증폭.
- 증폭 과정에서 cap 전하가 거의 0 으로 → **자동 rewrite (restoration)** 필요.

→ 그래서 row read 는 row 전체를 sense amp 로 펼치는 무거운 작업.

## 2. DRAM 어레이 구조

```
        BL0   BL1   BL2   ...   BL_n
         │     │     │           │
   WL0 ──C─────C─────C───────────C──     ← row 0 (수천 셀)
   WL1 ──C─────C─────C───────────C──     ← row 1
   ...
   WL_m──C─────C─────C───────────C──     ← row m
         │     │     │           │
        SA    SA    SA           SA      ← Sense Amplifier 행
         │     │     │           │
       col-mux (BL → IO)
```

- 한 row 당 보통 수천 셀 (8 KB ~ 16 KB row size).
- sense amp 가 row 안 모든 비트를 동시에 펼침.
- column 선택으로 그 row 안 특정 byte 만 IO 핀으로.

## 3. DRAM 동작 — Activate / R-W / Precharge

```
주소 도착
   │
   ▼
Activate (ACT, tRCD)  ← row 선택, sense amp 가 행을 활성 / restore
   │
   ▼
Read / Write (CL)     ← column 선택, data IO
   │
   ▼
(같은 row 안 R/W 는 ACT 없이 column burst 만 — 빠름)
   │
   ▼
Precharge (PRE, tRP)  ← sense amp 닫고 BL 을 Vdd/2 로 복귀
   │
   ▼
다음 row Activate
```

### 핵심 타이밍 (DDR4-3200 기준)

| 파라미터 | 의미 | 단위 | 일반값 |
| --- | --- | --- | --- |
| **tRCD** | Activate → 첫 Read | clk cycle | 22 (~13.75 ns) |
| **CL (CAS)** | Read 명령 → 데이터 도착 | clk cycle | 22 |
| **tRP** | Precharge → 다음 ACT | clk cycle | 22 |
| **tRAS** | ACT → 같은 row 의 PRE 최소 시간 | clk cycle | 52 |
| **tRC** | ACT → 같은 row 의 다음 ACT | clk cycle | tRAS + tRP = 74 |
| **tWR** | 마지막 Write → PRE | clk cycle | 24 |
| **tRRD** | 다른 bank 의 연속 ACT 최소 간격 | clk cycle | 8 |
| **tFAW** | 4 ACT 가 발생할 수 있는 최소 윈도우 | clk cycle | 32 |
| **tRFC** | refresh 사이클 시간 | ns | 350 |

### 실 ns 환산

- DDR5-6000, CL40 = `40 × (1/(6000/2 MHz)) = 40 × 0.333 = 13.3 ns`.
- DDR 의 "Double" 은 clock 의 양쪽 edge 모두 데이터 전송. clock 자체는 MT/s 의 절반.
- random access full cycle = `tRP + tRCD + CL = 22+22+22 = 66 cycle ≈ 41 ns` (DDR4-3200).

## 4. Page Hit / Miss / Conflict

| 상황 | 의미 | latency |
| --- | --- | --- |
| **Page Hit** | 원하는 row 가 이미 ACT 된 상태 | CL 만 (~ns) |
| **Page Miss** | 다른 row 가 ACT 되어 있음 (같은 bank) | PRE + ACT + CL |
| **Page Empty** | 그 bank 에 ACT 된 row 없음 | ACT + CL |
| **Bank Conflict** | 같은 bank 의 다른 row 연속 access | 항상 page miss |

→ 좋은 메모리 access 패턴 = sequential (page hit 위주) + bank interleaved.
→ random access pattern = page miss / bank conflict 빈발 → bandwidth 의 1/4 ~ 1/10 만 실효.

## 5. Refresh — 잊혀짐 방지

캐패시터 누설로 64 ms (DDR4 표준) 마다 모든 row 한 번씩 read+restore 필요.

### Auto Refresh (AR)

- 메모리 컨트롤러가 `REFRESH` 명령을 주기적으로 발행.
- 단일 REF = 보통 4096~16384 row 중 일부 (chip-internal counter 가 추적).
- 표준: 64 ms / 8192 row → 7.8 μs 에 1 REF.

### Self Refresh (SR)

- 시스템 idle (S3 sleep 등) 시.
- DRAM 칩이 자기 시계로 자체 refresh.
- 메모리 컨트롤러 off.

### Fine-Grained Refresh (FGR, DDR5)

- REF 를 더 짧게 / 잦게.
- 1x / 2x / 4x 모드. 더 잦을수록 throughput 영향 ↓.

### Refresh Management (RFM, DDR5)

- 메모리 컨트롤러가 명시적으로 특정 row 의 refresh 강제 (Rowhammer 대응).

### Refresh 의 영향

- refresh 가 진행 중인 bank/rank 는 ACT 불가 → throughput dip.
- tRFC 시간이 DDR 세대 / 용량 별 달라짐 (대용량일수록 길어짐). DDR5 16 Gb = ~350 ns.
- 데이터센터 16 GB DIMM 1 개의 refresh 부담 ~1% bandwidth.

## 6. Bank / Bank Group / Rank / Channel

### Bank
- DIMM 내부 독립 메모리 어레이.
- 각 bank 가 자기 sense amp 행 보유 → 다른 bank 끼리는 ACT 동시 가능.
- DDR4 = 16 bank/dimm. DDR5 = **32 bank/dimm**.

### Bank Group
- bank 의 묶음. 같은 group 안 연속 access 는 BG/CCD_L (long) 제약.
- 다른 group 끼리는 CCD_S (short) 제약 — 더 빠름.
- DDR5 = 8 bank group × 4 bank = 32 bank.

### Rank
- DIMM 위 chips 가 한 64-bit (또는 72-bit ECC) 워드를 만드는 묶음.
- 1R (single rank): chip 8 개 × 8-bit = 64-bit.
- 2R (dual rank): 같은 DIMM 안 두 묶음. 메모리 컨트롤러가 chip select 로 구분.
- 4R / 8R (LRDIMM): 더 많은 rank, register buffer 필요.

### Channel
- CPU 메모리 컨트롤러의 독립 경로.
- 데스크탑: dual channel (2 채널).
- 워크스테이션 (Threadripper, Xeon W): quad / octa.
- 서버: 8~12 채널 (AMD EPYC = 12, Intel Xeon = 8).

### bandwidth 정비례
- `total bandwidth = channel BW × channels`.
- 8 채널 EPYC 보드에 4 슬롯만 채우면 **BW 절반**.
- 채널 당 1 DIMM (1DPC) vs 채널 당 2 DIMM (2DPC) — 2DPC 는 보통 freq 조금 떨어짐 (signal integrity).

## 7. NUMA + DRAM

듀얼 소켓 / 멀티 다이 SoC = 각 소켓/다이가 자기 DRAM 보유.

```
Socket 0                    Socket 1
┌────────────┐              ┌────────────┐
│ Core 0-15  │              │ Core 16-31 │
│  IMC       │              │  IMC       │
└──┬─────────┘              └──┬─────────┘
   │ 8 channels                │ 8 channels
   │                           │
DRAM 0-7                     DRAM 8-15
   │                           │
   └─── Inter-socket fabric ───┘
        (Xeon UPI 또는
         EPYC Infinity Fabric)
```

- 로컬 메모리 (같은 소켓): 80~100 ns.
- 원격 메모리 (다른 소켓): 130~250 ns. **1.5–3 배 latency**.
- cross-socket bandwidth 도 제한 (UPI / IF 의 link).

```bash
numactl -H                  # NUMA 노드 / 메모리 분포
numactl --membind=0 --cpunodebind=0 ./app
lstopo --of console         # 시각적
```

### NUMA balancing (Linux)
- 자동: `/proc/sys/kernel/numa_balancing`.
- 작업 부하 측정 후 메모리/스레드 위치 이동.
- DB / VM / NCCL 등은 명시적 pinning 권장.

## 8. HBM (3D 적층 DRAM)

DRAM 다이 4–12 층을 TSV (Through-Silicon Via) 로 잇고 1024-bit 광폭 버스로 GPU 옆에 패키지 (CoWoS / 같은 interposer).

```
        Logic die (GPU)
         │
         │  silicon interposer (μ-bumps)
         │
   ┌─────┴─────┐
   │ HBM stack │  ← DRAM dies × 8~12
   │ TSV       │     이 stacks 4-6 개가 GPU 주변에 patterning
   │ TSV       │
   │ Base die  │  ← controller, IO
   └───────────┘
```

| 세대 | per-pin (Gbps) | per-stack BW (GB/s) | 대표 |
| --- | --- | --- | --- |
| HBM | 1 | 128 | Fiji (2015) |
| HBM2 | 2.4 | 307 | V100 |
| HBM2e | 3.6 | 460 | A100 |
| **HBM3** | 6.4 | 819 | H100 (80 GB, 3.35 TB/s 총 BW) |
| **HBM3e** | 9.6 | 1228 | H200, B100 (192 GB, ~8 TB/s) |
| HBM4 | 16+ | 2 TB/s+ | 2026~ 예정 |

### HBM 이 일반 DDR 을 대체 못 하는 이유
- 패키지 안 짧은 거리 + 첨단 패키징 필수 → 일반 보드 DIMM 슬롯에 못 들어감.
- 비싸다 (HBM 1 GB ≈ DDR5 1 GB 의 ~5 배).
- 용량 한계 (현 192 GB / GPU).

## 9. CXL.mem — Byte-Addressable Memory Pool

PCIe 5.0+ 의 physical layer 위에 캐시-일관성 프로토콜.

| 프로토콜 | 의미 |
| --- | --- |
| CXL.io | PCIe 호환 (enumeration / DMA) |
| CXL.cache | 가속기가 호스트 메모리를 캐시-일관성 R/W |
| **CXL.mem** | 호스트가 장치 메모리를 byte-addressable 로 사용 |

### Latency 비교

| 메모리 | latency |
| --- | --- |
| Local DDR (same socket) | 80–100 ns |
| Remote DDR (other socket) | 130–250 ns |
| CXL.mem (Type 3) | 200–300 ns |
| Persistent CXL (PCM/Optane 류) | 300–700 ns |
| NVMe random | 50–200 μs |

→ DRAM 보다 2-3 배 느리지만 NVMe 보다 100-1000 배 빠름. **새로운 메모리 티어**.

### 활용 시나리오 2025-2026
1. **메모리 풀링**: 여러 호스트가 한 CXL.mem 풀 공유. AI 학습 1 TB+ 메모리 요구.
2. **메모리 티어링**: hot = DRAM, warm = CXL.mem. Linux NUMA balancing 이 마이그레이션.
3. **메모리 익스팬더**: DDR 슬롯 채워도 한계 — CXL 익스팬더 카드로 +1 TB 가능.

## 10. Rowhammer — 인접 row 비트 플립

자세히는 [[ecc-rowhammer]] 참조. 본 노트 관점은 셀 물리.

- DRAM 행을 짧은 시간에 수십만 번 activate/precharge.
- 인접 row 캐패시터에 누설 전파 → 비트 플립.
- 셀 미세화 / 트랜지스터 누설 ↑ 로 갈수록 위험 ↑.
- 대응: **TRR** (DDR4), **RFM** (DDR5), **on-die ECC** (DDR5).

## 11. on-die ECC (DDR5)

- DDR5 DRAM 칩 안에 내부 ECC.
- 단일 비트 정정 / 검출 — chip 내부 노화 / 결함 대비.
- 단, DIMM ↔ 메모리 컨트롤러 사이 통신은 ECC 가 아님 (ECC RDIMM 별도).
- 진정한 end-to-end ECC = ECC RDIMM + chipkill 지원 컨트롤러.

## 12. DRAM 칩의 폼팩터 / 용량

| 종류 | 폭 | 일반 용량 (단일 칩) |
| --- | --- | --- |
| ×4 (x4) | 4-bit | 8–32 Gb (1–4 GB) |
| ×8 (x8) | 8-bit | 8–32 Gb |
| ×16 (x16) | 16-bit | 4–16 Gb |

DIMM 1 개의 용량 = chip 용량 × chip 수 / chip 폭 × 64-bit.

예: 8 Gb chip × 16 개 ÷ 8 = 16 GB. (2 rank × 8 chip)

## 13. 진단 / 측정

```bash
# DIMM 슬롯 / 속도 / ECC / 제조사
sudo dmidecode -t memory

# 라이브 통계
sudo apt install dstat
dstat -m 1
free -h
vmstat 1
mpstat -P ALL 1

# DRAM bandwidth 벤치마크
sudo apt install likwid
likwid-bench -t copy -w S0:8GB                # bandwidth, one socket
likwid-bench -t stream_avx512 -w S0:8GB

# Intel MLC (Memory Latency Checker)
sudo ./mlc --latency_matrix
sudo ./mlc --bandwidth_matrix

# EDAC (ECC) 카운터
sudo apt install edac-utils
edac-util -r
sudo cat /sys/devices/system/edac/mc/mc0/ce_count
```

## 14. 함정

1. **CL 만 비교** — 다른 MT/s 끼리 CL 숫자만 봐선 무의미. 실 ns 환산.
2. **bank conflict** — random access 가 같은 bank 의 다른 row 반복 → page miss 폭주.
3. **2DPC 가 1DPC 보다 항상 빠를 거라는 가정** — 실은 freq 다운그레이드로 더 느릴 수도.
4. **단일 채널 운영** — 듀얼 채널 보드에 1 모듈만 → BW 절반.
5. **모든 슬롯 비대칭 채우기** — channel A 만 채우고 B 비우면 dual channel 안 됨.
6. **DDR4 + DDR5 혼용 시도** — 물리 키 다름. 안 들어감.
7. **NUMA 무시한 컨테이너 / VM 배치** — cross-socket 메모리로 latency 폭증.
8. **on-die ECC 만 보고 안심** — DDR5 의 on-die ECC ≠ end-to-end ECC. RDIMM 별도.
9. **refresh dip 무시한 latency tail** — 99.9% 응답 latency 가 refresh 와 겹쳐 spike.
10. **HBM 용량 = DDR 대체 가정** — HBM 은 GPU/가속기 옆 짧은 거리 전용.

## 15. 관련

- [[memory]]
- [[memory-hierarchy]] — 캐시 ↔ DRAM ↔ 디스크
- [[ddr-evolution]] — DDR1~6 / LPDDR / HBM / CXL
- [[ecc-rowhammer]] — ECC, Chipkill, Rowhammer 변종
- [[../soc/soc-anatomy]] — NUMA / 메모리 컨트롤러
- [[../bus-io/pcie]] — CXL 의 physical layer
- [[../../operating-system/operating-system]] — huge page, NUMA balancing, swap
