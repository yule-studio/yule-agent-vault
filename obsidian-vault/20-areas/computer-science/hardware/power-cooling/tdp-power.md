---
title: "TDP 와 전력"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, power, tdp, pl1, pl2, tau, ppt, throttling, dvfs]
---

# TDP 와 전력

**[[power-cooling|↑ 전원·쿨링]]**

## 1. TDP — 정확한 의미

**Thermal Design Power**. 정격 부하에서 **쿨러가 처리해야 하는 열량 (W)**.

- ≠ 최대 전력 소비.
- ≠ 최저 전력 소비.
- 쿨링 / PSU 사이징 기준.

오해 1순위: "TDP 125 W CPU 는 항상 125 W 만 먹는다." → **틀렸음.**

## 2. 동적 전력 식

```
P_dynamic = α · C · V² · f
```

- α: 스위칭 활동률.
- C: 캐패시턴스.
- V: 전압.
- f: 주파수.

→ **전압 V 가 2 차로 영향.** DVFS (Dynamic Voltage Frequency Scaling) 이 전력을 줄이는 본질이 여기.

같은 freq 라도 V 를 5% 낮추면 P 가 ~10% 떨어짐 (undervolting).

## 3. 누설 전력

- 트랜지스터가 OFF 상태에서도 흐르는 전류 × V.
- 공정이 작아질수록 비중 ↑. 7 nm 이하 = 전체 전력의 30~50%.
- 클럭 / 활동 무관해서 idle 도 발열.

## 4. Intel — PL1 / PL2 / Tau

| 매개변수 | 의미 |
| --- | --- |
| **PL1 (Power Limit 1)** | 장기 평균 전력. 보통 TDP 와 같거나 같지 않음. |
| **PL2 (Power Limit 2)** | 짧은 부스트 한계. PL1 의 1.25~2.0×. |
| **Tau (τ)** | PL2 를 유지할 수 있는 시간 (s). 보통 28-56 s. |
| **PL3** | 1 ms 단위 spike. PSU 가 받아야 함. |
| **PL4** | 즉시 차단 한계. |

예: i9-14900K
- TDP 표기 125 W (PL1).
- PL2 = 253 W. Tau = 56 s.
- 단일 코어 부스트는 5.7-6.0 GHz, all-core 부스트는 5.5 GHz 근처.

## 5. AMD — PPT / TDC / EDC

| 매개변수 | 의미 |
| --- | --- |
| **PPT (Package Power Tracking)** | 전체 패키지 전력 한계 (W) |
| **TDC (Thermal Design Current)** | 평균 전류 한계 (A) |
| **EDC (Electrical Design Current)** | 짧은 spike 전류 한계 |
| **TjMax** | 최대 허용 junction 온도 (보통 95-105 °C) |

PBO (Precision Boost Overdrive) = 위 3 매개변수 풀어줘서 더 높은 부스트.

## 6. Thermal Throttling

- 코어 / 패키지 온도가 TjMax 도달 → 클럭 자동 다운.
- 1 회 1초 벤치 결과 ↔ 5 분 sustained 결과가 다른 이유.

| 단계 | 조치 |
| --- | --- |
| Tj 80 °C | 정상 |
| Tj 90 °C | 일부 boost 못 함 |
| Tj 95-100 °C | aggressive throttling |
| Tj > TjMax | shutdown |

### 측정
```bash
sensors
turbostat                         # Intel
powerstat / s-tui                 # 통합
```

## 7. GPU TGP / TBP

- **TGP (Total Graphics Power)** — GPU 칩만.
- **TBP (Total Board Power)** — VRAM / VRM / 팬 포함 카드 전체.
- RTX 4090 = TGP 450 W / TBP 450 W (대부분 같음).
- 데이터센터 H100 SXM5 = 700 W (configurable).

## 8. 노트북 / 모바일 차이

- **Cinematic mode** / **base mode** / **performance mode** — OS 가 PL1/PL2 동적 조절.
- 노트북 i7-13700H "45 W" 라벨이라도 PL2 115 W 단기 부스트.
- Apple M 시리즈는 PL 식별이 미공개 — sustained 와 burst 차이 작은 편.

## 9. Undervolting / Underclocking

- 전압을 정격 미만으로 → 같은 freq 에 전력 ↓.
- 안정성 한계까지 낮춘 후 stress test.
- 데스크탑 ↓ 효과 적음 (정격이 보수적). 노트북 / GPU 에서 흔히 -10~20% 전력.
- BIOS 또는 Intel XTU / AMD Ryzen Master / MSI Afterburner.

## 10. 함정

1. **"TDP 가 최대"** — 위 §1 의 가장 큰 오해. PL2 / PPT 까지 본 쿨링 / PSU.
2. **마더보드의 "no PL2 limit"** 설정 — 비용 줄이려고 항상 PL2 로 동작 → 발열 폭증.
3. **PL1 = TDP 가정** — Intel 은 다름. 마더보드가 PL1 을 임의로 올려놓은 경우 흔함.
4. **GPU TGP 만 보고 PSU 사이징** — transient spike 2× 까지.
5. **노트북 sustained 성능 평가 부재** — 1 분 후 sustained 가 burst 의 50–70%.
6. **undervolting 후 long-term 불안정** — 통과한 stress test 후에도 가끔 crash. 마진 두기.

## 11. 관련

- [[power-cooling]]
- [[psu-vrm]]
- [[../soc/soc-anatomy]]
