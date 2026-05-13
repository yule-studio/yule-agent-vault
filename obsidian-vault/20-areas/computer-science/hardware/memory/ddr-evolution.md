---
title: "DDR 세대 진화 (DDR / LPDDR / HBM / CXL)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, ddr, ddr4, ddr5, lpddr, hbm, cxl]
---

# DDR 세대 진화

**[[memory|↑ 메모리]]**

## 1. DDR (Double Data Rate)

데이터 라인이 **클록 양쪽 edge** (rising + falling) 에서 데이터를 전송 → 같은 clock freq 에서 2 배 throughput.

| 세대 | 출시 | 전압 | 표준 속도 (MT/s) | 채널 BW (GB/s) |
| --- | --- | --- | --- | --- |
| DDR1 | 2000 | 2.5 V | 200–400 | 1.6–3.2 |
| DDR2 | 2003 | 1.8 V | 400–1066 | 3.2–8.5 |
| DDR3 | 2007 | 1.5 V | 800–2133 | 6.4–17 |
| DDR4 | 2014 | 1.2 V | 1600–3200 | 12.8–25.6 |
| DDR5 | 2020 | 1.1 V | 4800–6400 (8000+ OC) | 38–51 |
| DDR6 | 2026~ | 1.0 V | 12800 (목표) | ~100 |

### bandwidth 환산

`bandwidth (GB/s) = MT/s × bus_width (byte) / 1000`

- DDR5-6400, 64-bit (8 byte): `6400 × 8 / 1000 = 51.2 GB/s/channel`.
- 듀얼 채널: 102.4 GB/s.
- 8 채널 EPYC: 410 GB/s.

## 2. DDR5 의 새 점

- **dual sub-channel**: 한 DIMM 이 32-bit × 2 sub-channel. 단일 DIMM 만으로도 일부 dual-channel 효과.
- **on-die ECC**: DRAM 칩 내부에서 비트 오류 자동 정정 (외부 ECC 와 다름).
- **PMIC (Power Management IC)** 가 모듈 위에. 전압 조절을 모듈이 직접.
- **fine-grained refresh** + **same-bank refresh** 옵션.

## 3. LPDDR (Low Power DDR)

모바일·노트북 (Apple Silicon, Snapdragon X Elite).

| 세대 | 전압 | 속도 (MT/s) |
| --- | --- | --- |
| LPDDR4X | 0.6 V | 4266 |
| LPDDR5 | 0.5 V | 6400 |
| LPDDR5X | 0.5 V | 8533 |
| LPDDR5T | 0.5 V | 9600 (Samsung) |
| LPDDR6 | (예정) | 14400+ |

특징:
- 패키지 내부 (PoP / SiP) 또는 솔더링 — 보통 교체 불가.
- 채널 폭이 다양 (x16 / x32 / x64) — Apple M 시리즈는 LPDDR5 채널 다수로 광 BW.

## 4. HBM (High Bandwidth Memory)

DRAM 다이 4–12 층을 TSV 로 잇고 1024-bit 광폭 버스로 GPU 옆에 패키지. CoWoS 같은 첨단 패키징과 한 묶음.

| 세대 | per-pin (Gbps) | per-stack | 대표 |
| --- | --- | --- | --- |
| HBM | 1 | 128 GB/s | Fiji (2015) |
| HBM2 | 2.4 | 307 GB/s | V100 |
| HBM2e | 3.6 | 460 GB/s | A100 |
| HBM3 | 6.4 | 819 GB/s | H100 |
| HBM3e | 9.6 | 1228 GB/s | H200, B100/B200 |
| HBM4 | 16+ | 2 TB/s+ | (예정 2026) |

- 짧은 거리·광폭이 핵심 — 일반 DIMM 대비 단일 stack 만 비교해도 30 배 BW.
- 단점: 비쌈. 패키지 면적 큼. 일반 데스크탑·서버용 DDR 을 대체하기엔 부적합 (가속기 전용).

## 5. CXL (Compute eXpress Link)

PCIe 5.0+ 의 physical layer 위에 캐시-일관성 프로토콜 3 종.

| 프로토콜 | 의미 | 예 |
| --- | --- | --- |
| **CXL.io** | PCIe 호환 (장치 enumeration / config) | 모든 CXL 장치 |
| **CXL.cache** | 가속기가 호스트 메모리를 캐시-일관성으로 read/write | GPU / SmartNIC |
| **CXL.mem** | 호스트가 가속기 / 메모리 익스팬더의 메모리를 byte-addressable 로 사용 | DRAM 풀, persistent memory |

2025-2026 본격 도입. 주요 배경:
- AI 학습 메모리 수요 폭증 — 단일 노드 1 TB+.
- DRAM 가격 폭등 (HBM·DDR5 수요).
- **memory pooling**: 여러 호스트가 한 DRAM 풀을 공유.
- **memory tiering**: hot 은 DRAM, warm 은 CXL.mem.

## 6. NVDIMM / Persistent Memory

- **NVDIMM-N**: DRAM + flash + 배터리. 정전 시 DRAM → flash dump.
- **NVDIMM-P / DCPMM (Intel Optane DC PMM)**: 본질적으로 PCM 기반 비휘발. **2022 Intel Optane 단종**.
- 후속은 CXL.mem (실 DRAM 또는 CXL-attached memory).

## 7. 함정

1. **CL 만 보고 비교** — 다른 MT/s 끼리 CL 만 비교는 의미 없음. 실제 ns 환산 후 비교.
2. **HBM = DRAM 대체 가정** — HBM 은 가속기 옆 짧은 거리 용. 데스크탑 / 서버 메인메모리 자리에 못 들어감.
3. **CXL.mem = DRAM 동일 속도 가정** — 일반 DDR DRAM 보다 latency 2–3 배. tiering 으로 hot/cold 구분 필수.
4. **세대 / 슬롯 혼동** — DDR4 ↔ DDR5 키 위치 다름.

## 8. 관련

- [[memory]]
- [[dram-cell]]
- [[ecc-rowhammer]]
- [[../soc/datacenter-accelerators]] — HBM 의 실 대표 적용처
- [[../bus-io/pcie]] — CXL 의 physical layer
