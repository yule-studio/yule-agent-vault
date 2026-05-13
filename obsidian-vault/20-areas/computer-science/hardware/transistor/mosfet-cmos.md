---
title: "MOSFET·CMOS 인버터"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, transistor, mosfet, cmos, nmos, pmos, finfet, gaa]
---

# MOSFET·CMOS 인버터

**[[transistor|↑ 트랜지스터·반도체]]**

## 1. MOSFET 기본

Metal-Oxide-Semiconductor Field-Effect Transistor.

- 4 단자: **gate (G)** / **drain (D)** / **source (S)** / **body/bulk (B)**.
- 게이트와 채널 사이는 얇은 산화막 (SiO₂ 또는 high-k) — 절연.
- 게이트 전압이 채널 안 캐리어를 끌어들여 source-drain 사이를 도통.

### nMOS 동작
- 채널은 전자.
- `V_GS > V_th` (threshold) → ON.
- 보통 V_th ~ 0.3–0.5 V (현세대).

### pMOS 동작
- 채널은 정공.
- `V_GS < -|V_th|` → ON.

## 2. CMOS 인버터

Complementary MOSFET — nMOS 와 pMOS 한 쌍.

```
         Vdd
          │
        ┌─┴─┐
        │pMOS│   ← gate = A
        └─┬─┘
          │── OUT = ¬A
        ┌─┴─┐
        │nMOS│   ← gate = A
        └─┬─┘
          │
         GND
```

| A | pMOS | nMOS | OUT |
| --- | --- | --- | --- |
| 0 (GND) | ON | OFF | Vdd (1) |
| 1 (Vdd) | OFF | ON | GND (0) |

전환 도중 짧은 순간 둘 다 ON → 관통 전류 (short-circuit current) → 동적 전력의 일부.

## 3. CMOS 의 매력

- **정적 누설 거의 0** (이상적): 두 트랜지스터 중 한 쪽이 OFF 라 source-drain 통전 없음.
- **fully-restored 전압**: 출력이 정확히 0 V / Vdd. 노이즈 마진 ↑.
- **대칭 + scalable**: NMOS only / PMOS only 비해 면적 ↑ 이지만 전력·신뢰성 ↑.

→ 1980 년대 후반부터 디지털의 표준.

## 4. CMOS NAND2 / NOR2

```
NAND2:
                Vdd
                 │
         ┌─┐  ┌─┐
         │P│  │P│   pMOS 2 병렬
         └┬┘  └┬┘
          └─┬──┘── Y
            │
         ┌──┴──┐    nMOS 2 직렬
         │ N   │
         ├─────┤
         │ N   │
         └──┬──┘
            │
           GND
```

→ Y = ¬(A·B). 트랜지스터 4 개.

NOR2 는 nMOS 병렬 / pMOS 직렬 — pMOS 직렬은 mobility 가 낮아 더 큰 트랜지스터 필요. 그래서 NAND 가 산업의 기본 셀.

## 5. 트랜지스터 진화 (planar → FinFET → GAAFET)

### Planar MOSFET (~22 nm 까지)
- 채널이 실리콘 위 평면.
- 작을수록 게이트 통제가 약해져 누설 폭증.

### FinFET (22 nm ~ 3 nm)
- 채널을 fin (지느러미) 으로 세움.
- 게이트가 3 면 (좌·우·위) 에서 감싸 통제력 ↑.
- Intel 22 nm (2012) 가 첫 양산. TSMC 16 nm (2014), Samsung 14 nm (2015).

### GAAFET / Nanosheet (3 nm 이하)
- 채널을 가로 시트 (nanosheet) 로, 게이트가 4 면 완전 감쌈.
- 시트 폭 조절로 transistor 강도 미세 조정 가능.
- Samsung 3GAE (2022), TSMC N2 (예정), Intel 18A.

### CFET (미래)
- nMOS·pMOS 를 수직 스택 → 표준 셀 면적 절반.

## 6. HKMG / SOI / Strain

세대 별 트랜지스터 강화 기법:

- **High-k Metal Gate (HKMG, 45 nm Intel 2007)** — SiO₂ 산화막을 HfO₂ 같은 high-k 로, 폴리실리콘 게이트를 metal 로. 누설 감소.
- **Strained Silicon (90 nm 부터)** — 실리콘 격자를 인장/압축 → 캐리어 이동도 ↑.
- **SOI (Silicon on Insulator)** — 절연막 위 실리콘. 누설 ↓, 일부 라이브러리 (IBM POWER) 가 채택.

## 7. 트랜지스터 전기적 모델 (간단)

- **선형 영역 (V_DS < V_GS - V_th)**: 저항처럼 작동.
- **포화 영역 (V_DS ≥ V_GS - V_th)**: 일정한 전류원.
- **subthreshold (V_GS < V_th)**: 누설 전류, 지수적으로 변화.

식 (단순 long-channel):
- 포화: `I_D = (μ·C_ox·W / 2L) · (V_GS - V_th)²`.
- 현세대는 short-channel + velocity saturation 으로 위 식이 deviating — 시뮬레이터 (SPICE) 필요.

## 8. 함정

1. **threshold voltage 가 고정이라는 가정** — 온도·공정·노화에 따라 변동.
2. **누설 무시** — 7 nm 이하부터 누설이 전체 전력의 30~50% 점유.
3. **NAND 와 NOR 가 면적 동등** — pMOS 의 mobility 가 낮아 NOR 면적이 더 큼.

## 9. 관련

- [[transistor]] — hub
- [[process-node-evolution]]
- [[../digital-logic/logic-gates]] — 위 단의 추상화
