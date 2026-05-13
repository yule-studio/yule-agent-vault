---
title: "공정 노드 진화 (Process Node Evolution)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, transistor, process-node, finfet, gaa, euv, moores-law, dennard]
---

# 공정 노드 진화 (Process Node Evolution)

**[[transistor|↑ 트랜지스터·반도체]]**

## 1. Moore's Law / Dennard Scaling — 그리고 둘의 종말

- **Moore's Law (1965, 1975 개정)** — 같은 가격의 칩에 들어가는 트랜지스터 수가 ~24 개월마다 2 배.
  - 물리 법칙이 아니라 **산업 로드맵 / 자기실현 예언**.
  - 2010 년대 중반부터 cadence 둔화 — 2 년 → 3 년 → 그 이상.
- **Dennard Scaling (1974)** — 트랜지스터를 미세화할 때 전력 밀도가 유지된다는 관찰.
  - 게이트 길이 / 산화막 두께 / 전압 / 전류를 모두 동일 비율로 줄이는 가정.
  - **2005~2006 년 사실상 종료** — 누설 전류와 양자 효과가 지배.
  - 그 결과 단일 코어 클럭이 3~5 GHz 박스에서 정체.

### 무엇으로 대체했나
1. **병렬화** — 멀티코어, SMT, GPU.
2. **특수화** — NPU, TPU, DSP, 비디오 코덱, 암호 가속기.
3. **3D 적층 / 첨단 패키징** — HBM, TSV, CoWoS, EMIB.

## 2. "공정 nm" 의 의미 변화

옛날 (1980-2000):
- "180 nm" = 게이트 길이 ≈ 180 nm.

현재 (2020+):
- "3 nm" 는 게이트 길이 자체가 아니다 — **마케팅 이름**.
- 실제 dimension 은 회사·세대마다 다르다.

진짜 비교 기준:

| 지표 | 의미 |
| --- | --- |
| **MMP (Metal Min Pitch)** | 가장 작은 메탈 라인 사이 간격 |
| **CPP (Contacted Poly Pitch)** | 인접 게이트 사이 간격 |
| **트랜지스터 밀도 (MTr/mm²)** | 1 mm² 에 박을 수 있는 트랜지스터 수 |

## 3. 세대 별 트랜지스터 밀도

| 공정 | 도입 시기 | 트랜지스터 밀도 (MTr/mm²) | 대표 칩 |
| --- | --- | --- | --- |
| 90 nm | 2003 | ~0.6 | Pentium 4 Prescott |
| 65 nm | 2005 | ~1 | Core 2 Duo |
| 45 nm | 2007 | ~2.5 | Penryn (HKMG 첫) |
| 32 nm | 2010 | ~7 | Westmere |
| 22 nm | 2012 | ~16 | Ivy Bridge (Intel FinFET 첫) |
| 14 nm | 2014 | ~37 | Broadwell, Samsung 14LPP |
| TSMC N7 | 2018 | ~91 | Apple A12, AMD Zen2, NVIDIA A100 |
| TSMC N5 | 2020 | ~173 | Apple A14/M1, Zen4 |
| TSMC N3E | 2023 | ~215 | Apple M3/M4, NVIDIA Blackwell |
| TSMC N2 (예정) | 2025-2026 | ~313 | Apple M5? |
| Intel 18A | 2025 | ~? (GAA + BSPDN) | Panther Lake |

> 같은 "5 nm" 라도 TSMC N5 ≈ Samsung 5LPE ≈ Intel 4 의 밀도는 모두 다르다.

## 4. 세대 별 트랜지스터 구조

| 구조 | 세대 | 핵심 차이 |
| --- | --- | --- |
| **Planar MOSFET** | ~22 nm | 평면 채널, 게이트가 한 면만 통제 |
| **FinFET** | 22 nm ~ 3 nm | 채널을 fin 으로, 게이트 3 면 통제 |
| **GAAFET / Nanosheet** | 3 nm 이하 | 채널을 시트로, 게이트 4 면 완전 감쌈 |
| **CFET** | 미래 (1 nm 이후?) | nMOS·pMOS 수직 스택 |

## 5. EUV (Extreme Ultraviolet) 노광

- 파장 13.5 nm. 기존 ArF immersion (193 nm) 보다 14 배 짧음.
- ASML 의 EUV scanner = 1 대 수백억 원, 진공 chamber, 액체 주석 droplet 을 CO₂ 레이저로 때려 광원 생성.
- 7 nm 부터 부분 EUV, 5 nm 부터 본격, 3 nm 이하 High-NA EUV.
- 1 wafer 노광 비용이 polynomial 로 증가 → 새 공정 비용이 폭증한 핵심 이유.

## 6. BSPDN (BackSide Power Delivery Network)

- 전통적으로 전원선과 신호선이 다이 앞면에 같이 깔림.
- BSPDN: 전원선을 다이 뒷면으로 분리.
- 결과: 신호선이 덜 막혀 면적 ↓, 전압 강하 ↓, 성능 ↑.
- Intel 20A / 18A (PowerVia), TSMC SPR (Super Power Rail, N2P 이후) 가 도입.

## 7. Cost / Yield 의 현실

- 7 nm wafer 가격 ≈ 14 nm 의 ~2 배.
- 5 nm 마스크 세트 가격 ≈ 5억 USD.
- 다이 크기 ↑ → 결함 누적 확률 ↑ → **수율 폭락**.
- 그래서 600 mm² 이상 단일 다이는 거의 사라지고, **칩렛 + 첨단 패키징** 이 정답이 됨.

## 8. 함정

1. **"공정 노드 작아지면 무조건 빨라진다"** — 누설·노이즈·발열 때문에 같은 freq 라도 효율은 세대 별로 천차만별.
2. **마케팅 nm 로 비교** — 트랜지스터 밀도 (MTr/mm²) 가 진짜 기준.
3. **양산 시점 = 발표 시점** — 발표 후 2~3 년 시제품, 양산까지 더 걸림.

## 9. 관련

- [[transistor]]
- [[mosfet-cmos]]
- [[../soc/datacenter-accelerators]] — 첨단 공정의 실 대표 적용처
