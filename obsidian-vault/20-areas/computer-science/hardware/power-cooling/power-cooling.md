---
title: "전원·쿨링 (Power / Cooling) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, power, psu, vrm, tdp, cooling, pue, immersion]
---

# 전원·쿨링 — Hub

**[[../hardware|↑ hardware]]**

## 0. 한 줄

**모든 트랜지스터는 전력을 먹고 열을 낸다. 데이터센터 시대의 진짜 병목은 클럭이 아니라 전원·쿨링이다.**

## 1. PSU / VRM
→ [[psu-vrm|PSU 와 VRM]]

ATX 12 V → VRM 1.0~1.4 V. 80 PLUS 인증 Bronze ~ Titanium. 12V-2x6 커넥터 (구 12VHPWR) 의 RTX 4090 600 W.

## 2. TDP / PL1 / PL2 / Tau
→ [[tdp-power|TDP 와 전력]]

TDP ≠ 최대 전력. Intel PL2 / Tau, AMD PPT 가 짧은 부스트.

## 3. 쿨링

| 방식 | 처리 W | 비고 |
| --- | --- | --- |
| 공랭 (대형 타워) | 200–300 W | Noctua, BeQuiet |
| AIO 수랭 | 250–400 W | 240/280/360/420 mm |
| 커스텀 수랭 | 400–600 W+ | 펌프 + 워터블록 + 라디 + 튜빙 |
| **DLC (Direct Liquid Cooling)** | kW급 | 서버 cold plate 직접 |
| **Immersion** | rack 단위 수십 kW | 단상 / 이상 (two-phase) |
| Vapor chamber | 모바일 / GPU | 평평한 heat pipe |

## 4. 데이터센터 지표

- **PUE (Power Usage Effectiveness)** = 총 전력 / IT 전력. hyperscale 1.1–1.2 가 정상.
- **WUE (Water Usage Effectiveness)** = L/kWh.
- **CUE (Carbon)** = kgCO₂/kWh.

## 5. AI 시대의 전력 / 쿨링

- DGX H100 ≈ 10.2 kW.
- GB200 NVL72 ≈ 120 kW per rack.
- 일반 데이터센터 rack 한계 (10-20 kW) 를 초과 → **DLC + 380/415 V DC + 4kW PSU** 필수.
- PUE 가 풍랭 1.4 → liquid 1.1 로 떨어짐.

## 6. 함정

1. **TDP 가 최대** 가정 → PL2 의 2× 전력 못 받아 throttling.
2. **PSU 80 PLUS 등급만 보고 W 부족** — 600 W RTX 4090 + 200 W CPU + 100 W 시스템 = 1000 W 권장 (transient 대응).
3. **12V-2x6 부분 체결** — 녹는 사고 빈번. 끝까지 딸각.
4. **공랭 / AIO 가 700 W GPU 감당** — 불가. DLC / immersion 필수.
5. **rack 단위 전력 분포 안 함** — 한 rack 만 전력 폭증 → 공조 부담.

## 7. 관련

- [[psu-vrm]]
- [[tdp-power]]
- [[../soc/datacenter-accelerators]]
