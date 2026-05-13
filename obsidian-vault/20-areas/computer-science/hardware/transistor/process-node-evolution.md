---
title: "공정 노드 진화 (Process Node Evolution)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, transistor, process-node, finfet, gaa, gaafet, euv, moores-law, dennard, bspdn, cfet, tsmc, intel, samsung]
---

# 공정 노드 진화 (Process Node Evolution)

**[[transistor|↑ 트랜지스터·반도체]]**

> 반도체 산업의 전체 로드맵 — Moore's Law / Dennard Scaling 의 시작에서 그 한계 / 첨단 패키징 / GAA 까지.

## 1. Moore's Law / Dennard Scaling — 시작과 종말

### Moore's Law (1965, 1975 개정)
- Gordon Moore (Intel 공동 창업자) 의 1965 논문 "Cramming more components onto integrated circuits".
- **같은 가격의 칩에 들어가는 트랜지스터 수가 ~24 개월마다 2 배**.
- 1975 에 18 → 24 개월로 보정.
- **물리 법칙이 아니라 산업 로드맵 / 자기실현 예언**.
- 2010 년대 중반부터 cadence 둔화 — 2 년 → 3 년 → 그 이상.

### Dennard Scaling (1974)
- IBM 의 Robert Dennard.
- 트랜지스터를 미세화할 때 전력 밀도가 유지된다는 관찰.
- 게이트 길이 / 산화막 두께 / 전압 / 전류를 모두 동일 비율로 줄이는 가정.

#### Dennard Scaling 의 식
- 트랜지스터 크기를 k 배 줄이면:
  - **delay** = 1/k 배 빠름.
  - **power per transistor** = 1/k² 배 감소.
  - **transistor density** = k² 배 증가.
  - **frequency** = k 배 ↑.
  - **power density (W/cm²)** = 그대로.

→ Moore's Law × Dennard Scaling = "더 작고 더 빠르고 더 효율"의 황금기 (1990s-2000s).

### Dennard Scaling 종말 (2005-2006)
- 누설 전류 (subthreshold + gate leakage) 가 동적 전력을 앞지르기 시작.
- 양자 효과로 게이트 산화막 더 못 얇아짐.
- 결과: **단일 코어 클럭이 3-5 GHz 박스에서 정체** (Pentium 4 의 4 GHz 한계).

### Moore's Law 도 둔화
- 28 nm → 14 nm: 4 년.
- 14 nm → 7 nm: 4 년.
- 7 nm → 5 nm: 2 년.
- 5 nm → 3 nm: 2 년.
- 3 nm → 2 nm: 3 년+ 예상.

## 2. 무어 / Dennard 종말 후 — 무엇으로 대체?

### 1. 병렬화 (2005-)
- Multi-core CPU.
- SMT / Hyper-Threading.
- GPU (Tesla 2006, CUDA 2007).

### 2. 특수화 (2010-)
- NPU (Apple Neural Engine, Google TPU).
- DSP (음성 / 비디오).
- 비디오 codec HW.
- AI 가속기 (H100 등).

### 3. 3D 적층 + 첨단 패키징 (2015-)
- HBM (Samsung 2013, AMD Fiji 2015).
- 3D NAND (Samsung V-NAND 2013).
- CoWoS (TSMC).
- 3D V-Cache (AMD 2022).

### 4. 새 트랜지스터 구조 (2012-)
- FinFET (Intel 22 nm, 2012).
- GAAFET / Nanosheet (Samsung 3GAE, 2022).

### 5. 새 인터커넥트 / 캐시 일관성
- NVLink / Infinity Fabric / CXL.

## 3. "공정 nm" 의 의미 변화

### 옛 (1980-2000)
- "180 nm" = 게이트 길이 ≈ 180 nm.
- 트랜지스터의 실 물리 dimension.

### 현재 (2020+)
- "3 nm" 는 게이트 길이 자체가 아니다 — **마케팅 이름**.
- 실제 dimension 은 회사 / 세대마다 다르다.
- 진짜 비교 기준이 필요:

| 지표 | 의미 |
| --- | --- |
| **MMP (Metal Min Pitch)** | 가장 작은 메탈 라인 사이 간격 |
| **CPP (Contacted Poly Pitch)** | 인접 게이트 사이 간격 |
| **트랜지스터 밀도 (MTr/mm²)** | 1 mm² 에 박을 수 있는 트랜지스터 수 |
| **SRAM 비트 밀도** | 1 mm² 당 SRAM 비트 |

## 4. 세대 별 트랜지스터 밀도

| 공정 | 도입 | 밀도 (MTr/mm²) | SRAM 밀도 (Mbit/mm²) | 대표 칩 |
| --- | --- | --- | --- | --- |
| 90 nm | 2003 | ~0.6 | — | Pentium 4 Prescott |
| 65 nm | 2005 | ~1 | — | Core 2 Duo, Cell |
| 45 nm | 2007 | ~2.5 | — | Penryn (HKMG 첫) |
| 32 nm | 2010 | ~7 | — | Westmere |
| 22 nm | 2012 | ~16 | — | Ivy Bridge (Intel FinFET 첫) |
| 14 nm | 2014 | ~37 | — | Broadwell, Samsung 14LPP |
| TSMC N10 | 2017 | ~52 | — | Apple A11 |
| **TSMC N7** | 2018 | **~91** | — | Apple A12, AMD Zen2, NVIDIA A100 |
| TSMC N6 | 2020 | ~115 | — | enhanced N7 |
| **TSMC N5** | 2020 | **~173** | ~27 | Apple A14/M1, Zen4 |
| TSMC N4 | 2022 | ~190 | — | optimized N5 |
| **TSMC N3E** | 2023 | **~215** | ~32 | Apple M3/M4, NVIDIA Blackwell |
| TSMC N3P | 2024 | ~225 | — | refined N3 |
| **TSMC N2** (예정) | 2025-2026 | **~313** | ~38 | Apple M5? |
| TSMC A16 (예정) | 2026 | ~350 | — | GAA + BSPDN |
| TSMC A14 (예정) | 2028 | ~430 | — | |
| **Intel 18A** | 2025 | ~? | — | GAA + BSPDN (PowerVia). Panther Lake. |

→ 같은 "5 nm" 라도 TSMC N5 ≈ Samsung 5LPE ≈ Intel 4 의 밀도는 모두 다르다.

### SRAM scaling 둔화
- 트랜지스터 밀도는 늘어나지만 SRAM scaling 은 N5 에서 멈춤.
- N5 → N3 = SRAM 거의 그대로 (~20% 만 ↑).
- → SoC 면적의 SRAM 비율이 커지면서 chiplet 전략이 더 중요.

## 5. 세대 별 트랜지스터 구조

| 구조 | 세대 | 핵심 차이 |
| --- | --- | --- |
| **Planar MOSFET** | ~22 nm | 평면 채널, 게이트가 한 면 통제 |
| **FinFET** | 22 nm ~ 3 nm | 채널을 fin (지느러미) 으로, 게이트 3 면 통제 |
| **GAAFET / Nanosheet** | 3 nm 이하 | 채널을 시트로, 게이트 4 면 완전 감쌈 |
| **CFET (Complementary FET)** | 미래 (1 nm 이후?) | nMOS·pMOS 수직 스택 |

자세히: [[mosfet-cmos]] §8.

## 6. FinFET 시대 (2012-2025)

### Intel 22 nm Tri-Gate (2012)
- 첫 양산 FinFET.
- 30% 성능 ↑, 50% 전력 ↓.
- Ivy Bridge.

### TSMC 16 nm FinFET (2014)
- Apple A9 / Snapdragon 820.
- 14 nm vs 16 nm — Samsung 14LPP 와 Intel 14 nm 가 더 정밀.

### TSMC 7 nm (2018)
- 첫 EUV 부분 도입.
- Apple A12, AMD Zen 2, NVIDIA A100.

### TSMC 5 nm (2020)
- EUV 본격.
- Apple A14 / M1, AMD Zen 4.

### TSMC 3 nm (2022 N3, 2023 N3E)
- EUV 더 적극.
- Apple M3 (N3B) — 수율 문제.
- Apple M4 / NVIDIA Blackwell — N3E.

## 7. GAAFET 시대 (2022+)

### Samsung 3GAE (2022)
- **첫 양산 GAAFET / MBCFET**.
- Samsung 4nm 와 비교한 ~17% 효율 ↑.
- 초기 수율 어려움 → 큰 고객 (Apple / NVIDIA) 안 가져옴.

### TSMC N2 (2025+)
- GAAFET 도입.
- N3E 대비 ~10-15% 성능 ↑, 25-30% 전력 ↓.
- Apple A19 Pro / M5 가 첫 고객 예상.

### Intel 20A / 18A (2024-2025)
- **20A** = RibbonFET (Intel 의 GAA) + PowerVia (BSPDN).
- **18A** = enhanced 20A. Panther Lake (2025).
- Intel 의 첫 modern 공정 — 18A 가 TSMC 와 경쟁.

### CFET (미래)
- 2030 후반 ~?
- nMOS·pMOS 수직 스택 → 표준 셀 면적 절반.

## 8. EUV (Extreme Ultraviolet) Lithography

### 기본
- **파장 13.5 nm**. 기존 ArF immersion (193 nm) 보다 14 배 짧음.
- 더 짧은 파장 = 더 작은 패턴 가능.
- **ASML** 만 만드는 1 대 ~150 M USD 장비.
- 진공 chamber, 액체 주석 droplet 을 CO₂ 레이저로 때려 광원 생성.

### EUV 단계별 도입
- TSMC 7nm+ (2019) — 부분 EUV.
- TSMC 5nm (2020) — 본격 EUV.
- TSMC 3nm (2022) — 더 많은 EUV layer.
- TSMC 2nm (2025) — High-NA EUV.

### High-NA EUV (NA 0.55)
- 더 큰 NA = 더 작은 feature.
- ASML Twinscan EXE:5000 (2024 첫 출하).
- 대당 ~350 M USD.
- Intel / TSMC / Samsung 모두 구매.

### EUV 의 비용
- 7 nm wafer 가격 ≈ 14 nm 의 ~2 배.
- 5 nm 마스크 세트 가격 ≈ 5 억 USD.
- → 새 공정 비용 폭증의 핵심 이유.

## 9. BSPDN (BackSide Power Delivery Network)

### 전통
- 전원선 + 신호선 모두 다이 앞면 metal layer.
- 미세화로 metal 가 빽빽 → 전원선이 신호선 막음 → IR drop ↑.

### BSPDN
- 전원선을 다이 **뒷면** 으로 분리.
- 앞면은 신호선만.
- 결과:
  - 신호 routing 면적 ↓.
  - IR drop ↓ (전원선이 두꺼움 가능).
  - 성능 ↑.

### 도입
- **Intel PowerVia** (20A, 2024).
- **TSMC Super Power Rail** (N2P, 2026+).
- **Samsung BSPDN** (3 nm 후속).

## 10. 첨단 패키징 — Chiplet + 3D 적층

### 왜 chiplet
- 큰 다이 (600 mm² +) = **수율 폭락**.
- 600 mm² 단일 다이 vs 100 mm² × 6 chiplet → 비용 절반.
- 다른 공정 mix 가능 (CPU = 5nm, IO = 12nm).
- yield ↑.

### TSV (Through-Silicon Via)
- 다이 관통 비아.
- 다이 위/아래 직접 연결.
- HBM 의 핵심.

### CoWoS (TSMC) — Chip-on-Wafer-on-Substrate
- silicon interposer + chiplet + HBM.
- NVIDIA H100/H200/Blackwell, AMD MI300X.
- 한 패키지 80×80 mm.

### EMIB (Intel) — Embedded Multi-die Interconnect Bridge
- 작은 실리콘 브릿지를 substrate 안에 매립.
- 다이 사이 짧은 거리 high-speed link.
- Intel Sapphire Rapids / Meteor Lake.

### 3D V-Cache (AMD)
- L3 캐시 die 를 CCD 위 적층 (TSV).
- Ryzen 7950X3D = 96 MB L3 (32 + 64 MB V-cache).
- 게이밍 / DB 의 working set 캐시.

### Apple UltraFusion (M Ultra)
- M Max 두 다이를 **silicon interposer** 로 결합.
- 2.5 TB/s die-to-die BW.
- 사용자에게는 단일 칩처럼 보임 (NUMA 약간).

## 11. Chiplet 아키텍처 — 산업 사례

### AMD Zen 2-5 / EPYC
- **CCD (Core Complex Die)** — 8 core, TSMC 5/7nm.
- **IOD (I/O Die)** — IO + memory controller, TSMC 12/6nm.
- EPYC 9004 (Genoa, 2022) = 12 CCD + 1 IOD = 96 core.
- EPYC 9005 (Turin, 2024) = 16 CCD + 1 IOD = 192 core (Zen 5c).

### Intel Meteor Lake (2023)
- 4 tile (chiplet):
  - **Compute tile** — Intel 4 (CPU + GPU).
  - **Graphics tile** — TSMC N5 (xe-LPG).
  - **SoC tile** — TSMC N6 (NPU, media, network).
  - **IO tile** — TSMC N6 (PCIe / Thunderbolt).

### Apple M Ultra
- 2 × M Max die.
- UltraFusion (silicon interposer + custom protocol).
- 2.5 TB/s.

## 12. UCIe — Die-to-Die 표준

PCIe 가 보드 위 인터커넥트라면 **UCIe 는 패키지 안 칩렛 사이**.

### UCIe 1.0 (2022)
- AMD / Intel / Samsung / TSMC / ARM / Google / Qualcomm / Microsoft / Meta / Alibaba 컨소시엄.
- short reach (≤ 25 mm) + standard.
- Tbps 단위 BW.

### UCIe 2.0 (2024)
- 더 빠른 link (32 GT/s per pin).
- coherency 옵션.

### UCIe 의 효용
- chiplet 표준화 → mix-and-match.
- 한 회사 chiplet + 다른 회사 chiplet 가능.

## 13. Yield / Cost 의 현실

### 다이 size vs yield
- defect density d (per mm²) 가정.
- yield ≈ (1 + Ad/k)^(-k) (Poisson model).
- A = 다이 면적.
- 600 mm² 다이 + d=0.1 → yield ~ 30%.
- 100 mm² × 6 chiplet → 각 yield ~ 90% → 6^0.9 = 53% (좀 부정확하지만 idea).

### Wafer cost
- 12-inch wafer:
  - 14 nm: ~$5,000.
  - 7 nm: ~$10,000.
  - 5 nm: ~$15,000.
  - 3 nm: ~$20,000.
  - 2 nm: ~$30,000+.

### Mask cost
- 5 nm 마스크 세트: ~$5 M.
- 3 nm: ~$10 M.
- 2 nm: ~$20 M+.
- → 적은 양산이면 마스크 비용 / wafer 가 압도.

## 14. 산업 경쟁 — 2024-2026

### TSMC
- 압도적 1 위. 5/3/2 nm.
- 거의 모든 AI gpu (NVIDIA / AMD / Cerebras / etc.) 의 fab.
- 가격 / 수율 / capacity 우위.

### Samsung Foundry
- 2 위. 3 GAAFET / 4 nm.
- 수율 어려움 (3GAE).
- Apple / NVIDIA / Qualcomm 의 고객 손실 → AMD / Intel / 자체 (Exynos) 만.

### Intel Foundry
- 3 위. Intel 4 / 18A.
- 2021 IDM 2.0 전략. Foundry 외판 시작.
- 18A 가 TSMC N2 와 경쟁 — 2025 양산.

### Chinese SMIC
- 미국 export 제재로 EUV 못 받음.
- DUV 만으로 7 nm 양산 (Huawei Kirin 9000s, 2023).
- 수율 / 비용 어려움.

## 15. 함정

1. **"공정 nm = 게이트 길이" 오해** — 실 dimension 은 마케팅명과 다름.
2. **마케팅 nm 로 비교** — 트랜지스터 밀도 (MTr/mm²) 가 진짜 기준.
3. **양산 시점 = 발표 시점** — 발표 후 2~3 년 시제품, 양산까지 더 걸림.
4. **공정 노드 작아지면 무조건 빨라진다** — 누설·노이즈·발열 때문에 같은 freq 라도 효율은 세대별로 천차만별.
5. **SRAM scaling 계속** 가정 — N5 이후 거의 멈춤. cache-heavy 칩의 면적 부담 ↑.
6. **HBM = DRAM 대체** — HBM 은 GPU 옆 짧은 거리 전용.
7. **single die scaling** — 600 mm² 이상은 yield 폭락. chiplet 필수.
8. **Intel 14nm 가 TSMC 14nm 와 동일** — Intel 14nm 가 더 dense.
9. **공정 발표 = 즉시 사용** — 신공정 첫해 capacity 가 매우 작음. 큰 고객만.
10. **EUV 없이 7 nm 가능** — SMIC 가 했지만 수율 / 비용 매우 나쁨.

## 16. 미래 — 2030+

### Beyond 1 nm
- CFET (Complementary FET).
- 2D 재료 (graphene, MoS2).
- carbon nanotube transistor.

### 새 paradigm
- Photonic computing.
- Quantum computing.
- Neuromorphic (memristive).
- DNA storage.

### 한계
- atom 크기에 가까워짐 — 한 셀 ~10 atom.
- 양자 효과 지배.
- thermal limit.

## 17. 관련

- [[transistor]]
- [[mosfet-cmos]]
- [[../soc/datacenter-accelerators]] — 첨단 공정의 실 대표 적용처
- [[../memory/dram-cell]] — DRAM 의 공정 scaling
- [[../memory/ddr-evolution]] — HBM 3D 적층
