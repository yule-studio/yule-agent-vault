---
title: "트랜지스터·반도체 (Transistor / Semiconductor) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, transistor, mosfet, cmos, finfet, gaa, process-node, packaging]
---

# 트랜지스터·반도체 — Hub

**[[../hardware|↑ hardware]]**

> 디지털 게이트가 실제 전자로 작동하는 단. 모든 위 층 (메모리·CPU·SoC) 의 비용·전력·속도의 천장.

## 0. 한 줄

**전압 신호 하나로 전류를 켜고 끄는 스위치 (트랜지스터) 를 nm 단위로 수십억 개 박은 칩.**

## 1. 트랜지스터 구조
→ [[mosfet-cmos|MOSFET·CMOS 인버터]]

- MOSFET (Metal-Oxide-Semiconductor FET) — 게이트 전압이 source-drain 채널을 개폐.
- nMOS / pMOS 의 상보 쌍이 CMOS — 정적 누설 거의 0 (이상적), 디지털 표준.

## 2. 트랜지스터 종류 / 진화

| 종류 | 시기 | 특징 |
| --- | --- | --- |
| Planar MOSFET | ~22 nm | 평면 채널 |
| **FinFET** | 22 nm ~ 3 nm | 채널을 지느러미로, 게이트가 3 면 감쌈 |
| **GAAFET / Nanosheet** | 3 nm 이하 | 채널을 시트로, 게이트가 4 면 감쌈 |
| **CFET** | 미래 | nMOS·pMOS 수직 스택 |

## 3. 공정 노드 (Process Node)
→ [[process-node-evolution|공정 노드 진화]]

- "7 nm / 5 nm / 3 nm" 는 더 이상 게이트 길이가 아닌 **마케팅 이름**.
- 진짜 비교는 **트랜지스터 밀도 (MTr/mm²)**, **MMP**, **CPP**.

| 공정 | 밀도 | 대표 칩 |
| --- | --- | --- |
| TSMC N7 | 91 MTr/mm² | Apple A12, Zen2 |
| TSMC N5 | 173 | Apple M1, Zen4 |
| TSMC N3E | 215 | Apple M3/M4 |
| TSMC N2 | ~313 | (예정) |

## 4. 전력 공식

- **동적 전력**: `P_dyn = α · C · V² · f`
  - 전압 V 가 2 차로 영향 — DVFS 의 본질.
- **누설 전력**: 트랜지스터 OFF 상태에서도 흐르는 전류. 공정이 작아질수록 비중 ↑.
- **Dennard scaling 종료** (2005-2006) → 단순 미세화로 전력당 성능 ↑ 가 멈춤.

## 5. EUV / 파운드리 / 팹리스

- **EUV (Extreme Ultraviolet)** — 13.5 nm 파장 노광. ASML 만 만드는 1 대 수백억 장비.
- **파운드리**: TSMC, Samsung Foundry, Intel Foundry, GlobalFoundries, SMIC.
- **팹리스**: Apple, NVIDIA, AMD, Qualcomm — 설계만, 제조는 위탁.
- **IDM**: Intel, Samsung, Micron — 설계+제조.

## 6. 첨단 패키징 / Chiplet

큰 다이 1 개 = 수율 ↓·비쌈. 작은 다이 여러 개를 한 패키지에 = 수율 ↑·조합 자유.

- **TSV (Through-Silicon Via)** — 다이 관통 비아.
- **CoWoS (TSMC)** — H100/Blackwell 의 GPU + HBM 결합.
- **EMIB (Intel)** — 작은 실리콘 브릿지.
- **3D V-Cache (AMD)** — L3 캐시 TSV 적층.

대표 칩렛 아키텍처:
- AMD Zen2~Zen5: CCD (코어) + IOD (IO).
- Intel Meteor Lake: 4 타일 (Compute / GPU / SoC / IO).
- Apple M Ultra: M Max 두 개를 UltraFusion 으로.

## 7. 함정

1. **"공정 nm = 게이트 길이" 오해** — 실 dimension 은 마케팅명과 다름.
2. **누설을 잊고 전력 모델 짜기** — 7 nm 이하부터 누설이 전력 절반에 근접.
3. **단일 다이 만능 가정** — 600 mm² 이상은 수율 폭락. 칩렛 + 패키징이 정답.

## 8. 관련

- [[../digital-logic/digital-logic]] — 게이트 단
- [[../memory/memory]] — SRAM/DRAM 셀의 트랜지스터 구조
- [[../soc/soc]] — 패키지·다이 토폴로지
