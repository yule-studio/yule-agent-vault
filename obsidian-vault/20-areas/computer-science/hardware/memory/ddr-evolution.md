---
title: "DDR 세대 진화 (DDR / LPDDR / HBM / CXL)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, ddr, ddr4, ddr5, ddr6, lpddr, hbm, cxl, jedec, pmic, bank-group]
---

# DDR 세대 진화

**[[memory|↑ 메모리]]**

> JEDEC 표준 DDR / LPDDR / HBM 의 세대별 진화. CXL.mem 의 새 티어까지.

## 1. DDR (Double Data Rate) 의 핵심

데이터 라인이 **클록 양쪽 edge** (rising + falling) 에서 데이터 전송 → 같은 clock freq 에서 2 배 throughput.

```
   Clock:  ─┐_┌─┐_┌─┐_┌─┐_┌─
   Data:   D1  D2 D3  D4 D5  D6  D7  ←  edge 마다 데이터
```

### MT/s vs MHz
- **MT/s** (MegaTransfers per second): 데이터 라인이 1 초에 토글하는 횟수.
- **MHz** (실 클록): MT/s 의 절반.
- DDR5-6400 = 6400 MT/s, 실 clock = 3200 MHz.

### bandwidth 환산

```
bandwidth (GB/s) = MT/s × bus_width (byte) / 1000
```

| 예 | 계산 |
| --- | --- |
| DDR5-6400, 64-bit (8 byte) per channel | `6400 × 8 / 1000 = 51.2 GB/s` |
| 듀얼 채널 | 102.4 GB/s |
| 8 채널 (EPYC) | 410 GB/s |
| 12 채널 (EPYC Genoa+) | 615 GB/s |

## 2. 세대별 사양

| 세대 | 출시 | 전압 | 표준 속도 (MT/s) | 채널 BW (GB/s) | 종료 |
| --- | --- | --- | --- | --- | --- |
| DDR1 | 2000 | 2.5 V | 200–400 | 1.6–3.2 | 단종 |
| DDR2 | 2003 | 1.8 V | 400–1066 | 3.2–8.5 | 단종 |
| DDR3 | 2007 | 1.5 V | 800–2133 | 6.4–17 | 노후 |
| **DDR4** | 2014 | 1.2 V | 1600–3200 | 12.8–25.6 | 주류 → 후퇴 중 |
| **DDR5** | 2020 | 1.1 V | 4800–6400 (8000+ OC) | 38–51 | 현 주류 |
| DDR6 | 2026~ | 1.0 V | 12800+ (목표) | ~100 | 사양 작업 중 |

### 전압 추이
- 1.5 V → 1.2 V → 1.1 V → 1.0 V.
- 더 낮은 전압 = 동적 전력 V² 로 감소.
- 데이터센터 전력 예산 핵심.

## 3. DDR5 — 가장 큰 구조적 변경

### Dual sub-channel
- 한 DIMM 의 64-bit 가 32-bit × 2 sub-channel 로 분리.
- 단일 DIMM 만으로도 dual-channel-like 동작.
- 메모리 컨트롤러가 두 sub-channel 독립 access 가능.

### Bank group 확장
- DDR4: 4 bank group × 4 bank = 16 bank.
- DDR5: 8 bank group × 4 bank = **32 bank**.
- 더 많은 bank → bank conflict 감소 → 효율 ↑.

### Burst length
- DDR4: BL8 (8 cycle burst, 64 byte at 64-bit).
- DDR5: BL16 (16 cycle burst, 64 byte at 32-bit sub-channel).
- 같은 cache line (64 byte) 을 동일하게 처리.

### On-die ECC
- 각 DDR5 DRAM 칩 안에 internal ECC.
- 128-bit data + 16-bit ECC.
- 단일 비트 자동 정정.
- **단, DIMM ↔ 메모리 컨트롤러 사이 통신은 ECC 가 아님**. 진정한 end-to-end ECC = ECC RDIMM 별도.

### PMIC (Power Management IC)
- 옛 DDR4: 메인보드가 1.2 V 공급.
- DDR5: 모듈 위 PMIC 가 1.1 V 자체 생성.
- VRM 노이즈 격리 + 모듈 별 정밀 조절.
- OC 시 PMIC 의 phase / current limit 가 천장.

### Fine-grained refresh + RFM
- 자세히: [[dram-cell]] §5.
- 1x / 2x / 4x refresh 모드.
- Rowhammer 대응 RFM (Refresh Management).

### CAMM2 폼팩터
- 노트북용. LPDDR5X + DDR5 를 통합.
- M.2 비슷한 평평한 모듈 + 한 면에 칩.

## 4. DDR5 채택 추이

| 플랫폼 | DDR5 시작 |
| --- | --- |
| Intel | Alder Lake (12th gen, 2021) — DDR4/5 둘 다. Raptor Lake (13th, 2022) — 동일. Meteor Lake (14th, 2024) — DDR5 only on mobile. Arrow Lake (Core Ultra, 2024) — DDR5 only desktop. |
| AMD | AM5 (Ryzen 7000+, 2022) — DDR5 only. EPYC Genoa (4th gen, 2023) — DDR5 12 채널. |
| Apple | M1 (2020) — LPDDR4X. M2 — LPDDR5. M3/M4 — LPDDR5/5X. |
| ARM 서버 | Graviton 3 (2022) — DDR5. Graviton 4 (2024). |

## 5. LPDDR — 모바일·노트북

| 세대 | 전압 | 속도 (MT/s) | 출시 |
| --- | --- | --- | --- |
| LPDDR3 | 1.2 V | 1866 | 2012 |
| LPDDR4 | 1.1 V | 4266 | 2014 |
| LPDDR4X | 0.6 V | 4266 | 2017 |
| **LPDDR5** | 0.5 V | 6400 | 2019 |
| LPDDR5X | 0.5 V | 8533 | 2021 |
| LPDDR5T | 0.5 V (Samsung) | 9600 | 2023 |
| LPDDR6 | (예정) | 14400+ | 2026~ |

### 특징
- 매우 낮은 전압 (0.5 V) → idle 전력 ↓.
- 패키지 내부 (PoP / SiP) 또는 솔더링 — 보통 교체 불가.
- 채널 폭이 다양 (x16 / x32 / x64) — Apple M 시리즈는 LPDDR5 채널 다수로 광 BW.

### Apple Silicon 의 메모리 BW

| 칩 | 채널 폭 | BW |
| --- | --- | --- |
| M1 | 128-bit | 68 GB/s |
| M1 Pro | 256-bit | 200 GB/s |
| M1 Max | 512-bit | 400 GB/s |
| M1 Ultra | 1024-bit | 800 GB/s |
| M2 | 128-bit | 100 GB/s |
| M2 Max | 512-bit | 400 GB/s |
| M2 Ultra | 1024-bit | 800 GB/s |
| M3 Max | 512-bit | 400 GB/s |
| M4 Max | 512-bit | 546 GB/s |

→ M Ultra 의 800 GB/s = 일반 데스크탑 DDR5 의 ~8 배. LLM 추론 / Final Cut 같은 메모리 BW bound 워크로드의 큰 장점.

### LPDDR vs DDR 의 운영 차이
- LPDDR = 솔더링 → 교체 불가, 업그레이드 불가.
- 양산 시 칩 grade 고정 — 같은 모델이라도 다른 메모리 grade 못 받음.

## 6. HBM (High Bandwidth Memory) — 3D 적층

DRAM 다이 4–12 층을 TSV 로 잇고 1024-bit 광폭 버스로 GPU 옆에 패키지.

```
        Logic die (GPU controller)
         │
   ┌─────┴─────┐
   │ DRAM 8/12 │  ← die stack
   │ TSV       │
   │ TSV       │
   │ Base die  │  ← logic / IO
   └───────────┘
         │
   silicon interposer (μ-bumps)
         │
   ┌─────┴─────┐
   │   GPU     │
   └───────────┘
```

### 세대

| 세대 | per-pin (Gbps) | per-stack BW (GB/s) | per-stack 용량 | 대표 |
| --- | --- | --- | --- | --- |
| HBM | 1 | 128 | 4 GB | Fiji (2015) |
| HBM2 | 2.4 | 307 | 8–16 GB | V100 (2017) |
| HBM2e | 3.6 | 460 | 16 GB | A100 |
| **HBM3** | 6.4 | 819 | 16–24 GB | H100 (80 GB, 3.35 TB/s) |
| **HBM3e** | 9.6 | 1228 | 24–36 GB | H200, B100 (192 GB, ~8 TB/s) |
| HBM4 | 16+ | 2 TB/s+ | 32–64 GB | 2026~ 예정 |
| HBM4e | (예정) | 더 ↑ | 더 ↑ | 2027 |

### CoWoS / 첨단 패키징
- TSMC CoWoS (Chip-on-Wafer-on-Substrate) 가 H100/Blackwell GPU + HBM 결합의 표준.
- 큰 silicon interposer 위 GPU + 4-6 HBM stack.
- 패키지 면적 ~80×80 mm.

### HBM 이 DDR 을 대체 못 하는 이유
- **짧은 거리 + 첨단 패키징 필수** — 일반 DIMM 슬롯에 못 들어감.
- **비쌈** — HBM 1 GB ≈ DDR5 1 GB 의 ~5 배.
- **용량 한계** — 현 192 GB/GPU. DDR 서버는 4 TB+ 가능.
- → GPU / 가속기 전용. CPU 서버는 DDR.

### HBM 공급망
- Samsung / SK hynix / Micron 3 사 과점.
- 2024-2025 HBM3e 의 NVIDIA 의 모든 capacity 예약 → 다른 회사 부족.

## 7. CXL (Compute Express Link) — PCIe 위의 캐시 일관성

PCIe 5.0+ 의 physical layer 위에 3 종 프로토콜.

### 프로토콜 3 종

| 프로토콜 | 의미 | 장치 type |
| --- | --- | --- |
| **CXL.io** | PCIe enumeration + DMA | 모든 CXL 장치 |
| **CXL.cache** | 가속기가 호스트 메모리 캐시-일관성 R/W | Type 1 (cache only) / Type 2 |
| **CXL.mem** | 호스트가 장치 메모리를 byte-addressable | Type 2 / Type 3 |

### 장치 type

| Type | 가속기 캐시 | 자체 메모리 | 예 |
| --- | --- | --- | --- |
| **Type 1** | ✓ | ✗ | SmartNIC, accelerator |
| **Type 2** | ✓ | ✓ | GPU / 가속기 (Intel Ponte Vecchio 등) |
| **Type 3** | ✗ | ✓ | DRAM 익스팬더, persistent memory |

### Latency 비교

| 메모리 티어 | latency | 용도 |
| --- | --- | --- |
| Local DDR (same socket) | 80–100 ns | hot working set |
| Remote DDR (other socket) | 130–250 ns | NUMA |
| **CXL.mem (Type 3, near)** | 170–200 ns | warm extended memory |
| **CXL.mem (Type 3, far)** | 250–300 ns | larger pool |
| Persistent CXL (PCM/Optane 류) | 300–700 ns | persistent tier |
| NVMe random | 50–200 μs | cold |

→ DRAM 보다 2-3 배 느리지만 NVMe 보다 100-1000 배 빠름. **새로운 메모리 티어**.

### CXL 1.1 vs 2.0 vs 3.0

| 버전 | 주요 기능 | 출시 |
| --- | --- | --- |
| CXL 1.1 | direct-attach memory | 2020 |
| CXL 2.0 | switching, memory pooling 시작 | 2021 |
| CXL 3.0 | fabric, multi-headed devices, multiple hosts | 2023 |
| CXL 3.1 | direct peer-to-peer | 2024 |

### 활용 시나리오 2025-2026

1. **메모리 풀링** — 여러 호스트가 한 CXL.mem 풀 공유. AI 학습 1 TB+ 메모리 요구.
2. **메모리 티어링** — hot = DRAM, warm = CXL.mem. Linux NUMA balancing 마이그레이션.
3. **메모리 익스팬더** — DDR 슬롯 다 채우고도 +1 TB CXL 익스팬더.
4. **disaggregated memory** — 메모리를 컴퓨트와 분리한 chassis.

### CXL switch / fabric

- CXL 2.0+ 의 switch.
- 여러 host 가 같은 CXL.mem 장치 다중 binding.
- 데이터센터의 메모리 분리 (storage-like disaggregation).

### 실 CXL 제품

| 제품 | 종류 | 비고 |
| --- | --- | --- |
| Samsung CXL Memory Module | Type 3 | 512 GB DDR5 익스팬더 |
| Micron CZ120 | Type 3 | 128/256 GB |
| Astera Labs Leo | Type 3 / switch | CXL 2.0 controller |
| Intel Granite Rapids / AMD Genoa+ | host | CXL 2.0 host 지원 |

## 8. NVDIMM / Persistent Memory

### NVDIMM-N
- DRAM + flash + 배터리.
- 정전 시 DRAM → flash dump.
- 큰 용량 + DRAM 속도.
- 비쌈 + 배터리 관리.

### NVDIMM-P / DCPMM (Intel Optane DC PMM)
- 본질적 비휘발 매체 (PCM).
- 메모리 슬롯에 꽂아 byte-addressable persistent memory.
- 2022 Intel Optane 단종 → CXL 로 대체 흐름.

### 후속
- CXL.mem 위 persistent memory.
- KIOXIA / SK hynix / Samsung 의 PCM / FeRAM / MRAM 일부 시제품.

## 9. 메모리 압축 / 중복 제거

### Memory Compression (Linux zswap, macOS)
- swap-out 대신 메모리 안에서 압축.
- LZ4 / Zstandard.
- 2-3× 효과적 메모리.

### KSM (Kernel Samepage Merging)
- 같은 내용 page 를 한 물리 page 로 병합.
- VM 호스트의 같은 OS guest 들 사이 효과 크다.

### Memory Deduplication (VMware TPS)
- 비슷한 메커니즘.
- 보안 (사이드채널) 이유로 기본 off.

## 10. 메모리 컨트롤러 — IMC

옛 Northbridge 의 메모리 컨트롤러가 CPU 안으로 통합.

### CPU 내장 IMC
- Intel: Nehalem (2008) 부터.
- AMD: Athlon 64 (2003) 부터 (AMD 가 먼저).
- Apple Silicon: 처음부터.

### IMC 의 역할
1. DDR 명령 발행 (ACT/RD/WR/PRE).
2. Refresh 자동 처리.
3. ECC 계산 / 검증.
4. Bank scheduling (page hit 최적화).
5. NUMA - 자기 소켓 IMC 가 자기 DRAM 책임.

### IMC 의 다중 채널 scheduling
- 8 채널 IMC 가 들어오는 요청을 hash → 8 채널 분산.
- 같은 cache line 의 burst 는 한 채널 안에서 처리.

## 11. 진단 / 측정

```bash
# DIMM 슬롯 / 속도 / ECC / 제조사
sudo dmidecode -t memory
sudo dmidecode -t 17                # type 17 = memory device

# 라이브 통계
sudo apt install dstat
dstat -m 1
free -h
vmstat 1

# DRAM bandwidth 벤치마크
sudo apt install likwid
likwid-bench -t copy -w S0:8GB                # 한 소켓
likwid-bench -t stream_avx512 -w S0:8GB

# Intel MLC (Memory Latency Checker)
sudo ./mlc --latency_matrix
sudo ./mlc --bandwidth_matrix
sudo ./mlc --idle_latency

# 채널 별 BW
sudo apt install perfsuite
perfsuite -t mem-bw

# NUMA
numactl -H
numastat -m

# memtier
sudo apt install memtier_benchmark
```

## 12. OC / 튜닝 — 데스크탑

### XMP / EXPO
- **XMP (Intel)** — DIMM 의 SPD 에 미리 정의된 OC profile.
- **EXPO (AMD)** — AMD 의 동급 표준.
- DDR5-6000 CL30 같은 패키지가 보통 XMP/EXPO 1 profile 보유.

### Primary timing
- CL / tRCD / tRP / tRAS — 가장 큰 영향.

### Secondary / Tertiary timing
- tRFC, tRRD_S/L, tFAW, tWR, tWTR, tCWL, etc.
- AMD AGESA / Intel ME 의 자동 / 수동 설정.

### 안정성 테스트
- TestMem5 (Anta 777 prefix).
- Y-Cruncher VST/HNT.
- MemTest86 24 시간 이상.

## 13. 함정

1. **CL 만 보고 비교** — 다른 MT/s 끼리 CL 만 봐선 의미 없음. 실 ns 환산 필수.
2. **HBM = DDR 대체** — HBM 은 GPU 옆 짧은 거리 전용. CPU 서버 메모리 자리 못 들어감.
3. **CXL.mem = DRAM 동일 속도** — 2-3 배 느림. tiering 으로 hot/cold 구분 필수.
4. **세대 혼용 시도** — DDR4 ↔ DDR5 키 위치 다름. 안 들어감.
5. **모든 슬롯 다 채우면 더 빠름** — 2DPC 는 보통 freq 조금 떨어짐 (signal integrity).
6. **XMP 켜면 항상 안정** — 일부 보드 / CPU 조합에서 4 channel 동시 XMP 가 fail.
7. **on-die ECC 만 보고 안심** — DDR5 의 on-die ECC ≠ end-to-end ECC.
8. **LPDDR 모듈 교체 시도** — 솔더링이라 불가.
9. **HBM3e 가용성** — NVIDIA 가 거의 모두 예약. 다른 회사 부족 → 가격 폭등.
10. **CXL.mem 의 NUMA 노드 인식** — Linux 가 CXL device 를 별도 NUMA 노드로 expose. 응용이 의식해야 효과.

## 14. 관련

- [[memory]]
- [[memory-hierarchy]] — 계층 위 위치
- [[dram-cell]] — DRAM 1T1C 셀 동작
- [[ecc-rowhammer]] — DDR5 on-die ECC + Rowhammer
- [[../bus-io/pcie]] — CXL 의 physical layer
- [[../soc/datacenter-accelerators]] — HBM 실 적용처
- [[../soc/soc-anatomy]] — Apple Silicon UMA
- [[../../operating-system/operating-system]] — NUMA balancing, huge page, zswap
