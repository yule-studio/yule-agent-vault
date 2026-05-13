---
title: "MOSFET·CMOS 인버터"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, transistor, mosfet, cmos, nmos, pmos, finfet, gaa, hkmg, body-effect, subthreshold]
---

# MOSFET·CMOS 인버터

**[[transistor|↑ 트랜지스터·반도체]]**

> 디지털의 가장 작은 단위. 모든 게이트·메모리 셀·캐시·CPU 가 이 트랜지스터들의 조합.

## 1. MOSFET 기본 구조

```
     Source ──┐    Gate (절연된)    ┌── Drain
              │    │                │
              ▼    ▼                ▼
            ┌──────────────────────────┐
            │  n+   ─ Oxide ─    n+    │  ← p-type substrate (nMOS)
            │  doping  thin    doping  │
            │            ↑             │
            │            channel       │
            │       (전기장으로 형성)   │
            └──────────────────────────┘
                    │
                    ▼ (substrate / body / bulk)
```

### 4 단자
- **Gate (G)** — 채널 전기장 제어. 절연막 (SiO₂ 또는 high-k) 을 사이에 두고 채널과 분리.
- **Source (S)** — 전자/정공의 공급원.
- **Drain (D)** — 전자/정공의 도착지.
- **Body / Bulk (B)** — substrate. 보통 source 와 같은 전압 (tied to source).

### 종류

| 종류 | 채널 캐리어 | substrate | ON 조건 |
| --- | --- | --- | --- |
| **nMOS** | 전자 | p-type | V_GS > V_th (> 0) |
| **pMOS** | 정공 | n-type | V_GS < -|V_th| |

## 2. 동작 영역

전압별 동작:

| 영역 | 조건 (nMOS) | 동작 |
| --- | --- | --- |
| Subthreshold | V_GS < V_th | OFF (누설 전류만 흐름) |
| Linear / Triode | V_GS > V_th, V_DS < V_GS−V_th | 저항처럼 동작 |
| Saturation | V_GS > V_th, V_DS ≥ V_GS−V_th | 전류원처럼 동작 |

### 식 (long-channel 단순 모델)

**Linear 영역**:
```
I_D = μ_n · C_ox · (W/L) · ((V_GS − V_th) · V_DS − V_DS²/2)
```

**Saturation 영역**:
```
I_D = (μ_n · C_ox / 2) · (W/L) · (V_GS − V_th)² · (1 + λ·V_DS)
```

- μ_n: 전자 이동도.
- C_ox: 산화막 캐패시턴스 (단위 면적).
- W/L: 채널 width/length 비.
- λ: channel-length modulation.

→ **W/L 가 트랜지스터 강도의 핵심 design knob**. 표준 셀 라이브러리는 같은 게이트 종류를 X1 / X2 / X4 / X8 의 drive strength 변종으로 제공.

### Short-channel effects

현대 nm 공정에서는 short-channel 효과로 위 식이 deviating:
- **Velocity saturation**: 캐리어 속도 한계.
- **DIBL** (Drain-Induced Barrier Lowering): V_DS 가 threshold 끌어내림.
- **Hot carriers**: 고에너지 캐리어가 산화막 손상.

→ SPICE 등 회로 시뮬레이터의 BSIM 모델 (40+ 파라미터) 필요.

## 3. 핵심 매개변수

| 파라미터 | 의미 | 일반값 (28 nm 기준) |
| --- | --- | --- |
| **V_th (Threshold voltage)** | 채널 형성 임계 전압 | 0.3 ~ 0.5 V |
| **V_dd (Supply voltage)** | 회로 전원 | 0.9 ~ 1.0 V |
| **t_ox** | 산화막 두께 | 1 ~ 2 nm (실제 두께) |
| **L** | 채널 길이 | 28 nm (마케팅) |
| **W** | 채널 폭 | design 별 |
| **μ_n / μ_p** | 이동도 비율 | ~2.5 (electron > hole) |

### 왜 μ_p < μ_n
- 정공의 이동도가 전자의 ~40%.
- pMOS 가 같은 전류 내려면 nMOS 의 ~2-2.5 배 W.
- → CMOS 회로의 pMOS 가 보통 nMOS 보다 큼.

## 4. CMOS 인버터 — 디지털의 시작

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

### 동작 진리표
| A | pMOS | nMOS | OUT |
| --- | --- | --- | --- |
| 0 (GND) | ON | OFF | Vdd (= 1) |
| 1 (Vdd) | OFF | ON | GND (= 0) |

### 전이 (transition) — 짧은 관통 전류

A 가 GND → Vdd 변환 중간 구간 (~Vdd/2 근처) 에서 pMOS / nMOS 둘 다 잠시 ON → source-drain 직접 short circuit → **관통 전류 (short-circuit current)**.

- 한 transition 당 짧지만 ns 단위 발생.
- 동적 전력의 ~10-20% 차지.
- 빠른 transition slope 가 관통 전류 ↓.

## 5. 동적 / 정적 / 누설 전력

### 동적 전력 (스위칭 시)

```
P_dyn = α · C_L · V_dd² · f
```

- α: 스위칭 활동률 (0~1).
- C_L: load 캐패시턴스.
- V_dd: 전원.
- f: 클럭 주파수.

→ **V_dd 가 2 차로** 영향. 1 V → 0.95 V (5% ↓) 면 P ≈ 10% ↓ (DVFS 의 본질).

### 정적 / 누설 전력

스위칭 안 해도 흐르는 전류:

| 종류 | 의미 |
| --- | --- |
| **Subthreshold leakage** | OFF 상태에서 V_GS < V_th 의 작은 전류. exponential 의존 V_th. |
| **Gate leakage** | 게이트 산화막을 직접 통과하는 양자 터널링. t_ox 얇을수록 ↑. |
| **GIDL (Gate-Induced Drain Leakage)** | drain↔body 사이 누설. |
| **Junction leakage** | reverse-biased p-n 접합 누설. |

→ 7 nm 이하에서 누설이 전체 전력의 **30~50%** 차지.

### HKMG (High-k Metal Gate) — 누설 잡기

- 옛 SiO₂ + 폴리실리콘 게이트 → high-k 절연체 (HfO₂ 등) + metal 게이트.
- 같은 전기적 두께 (EOT) 면서 물리적으로 두꺼움 → 양자 터널링 ↓.
- Intel 45 nm (2007) 부터, TSMC 28 nm (2011) 부터 양산.

## 6. CMOS NAND2 / NOR2 — 표준 셀

### NAND2 (Y = ¬(A · B))

```
            Vdd
        ┌────┴─────┐
       pA│        │pB     pMOS 2 병렬
        ┴          ┴
        ────┬──┬──── 
            │  │
            └──┘── Y
            ┌──┐
        ────┴──┴────
        ┬          ┬
       nA│        │nB     nMOS 2 직렬
        └────┬─────┘
            GND
```

- pMOS 2 개 **병렬** (둘 중 하나만 ON 이어도 Y = Vdd).
- nMOS 2 개 **직렬** (둘 다 ON 이어야만 Y = GND).
- 트랜지스터 4 개. delay 가 NOR 보다 짧음.

### NOR2 (Y = ¬(A + B))

- pMOS 2 개 **직렬** (둘 다 ON 이어야 Y = Vdd).
- nMOS 2 개 **병렬** (둘 중 하나만 ON 이어도 Y = GND).
- pMOS 직렬은 mobility ↓ 때문에 더 큰 트랜지스터 → 큰 면적 + 큰 delay.

### 왜 NAND 를 더 많이 쓰는가
1. 같은 fan-in 에서 NAND 가 빠르고 작다.
2. NAND universality (NAND 만으로 모든 회로 구성).
3. 산업 표준 셀 라이브러리의 기본 셀.

## 7. CMOS 의 fan-out / drive strength

### Fan-out

게이트 출력이 끌어야 하는 다음 입력 수.

- 각 다음 입력 = gate capacitance.
- 더 많은 fan-out = 더 큰 load capacitance = 더 긴 delay.

### Drive strength (X1, X2, X4, X8 …)

- 같은 NAND2 라도 트랜지스터 width 가 다른 변종.
- 큰 X = 강한 drive = 빠르지만 면적·전력 ↑.

```
NAND2_X1 → 가벼운 fan-out 1-2 개 용
NAND2_X4 → 무거운 fan-out 5-10 개 용
NAND2_X16 → 큰 bus driver
```

합성 (synthesis) 도구가 path 의 timing 요구에 맞춰 자동 선택.

## 8. 트랜지스터 진화 — Planar → FinFET → GAAFET → CFET

### Planar MOSFET (~22 nm 까지)

- 채널이 실리콘 위 평면.
- 게이트가 한 면에서만 통제 → short-channel 효과 폭증.

### FinFET (22 nm ~ 3 nm)

- 채널을 "fin" (지느러미) 으로 세움.
- 게이트가 **3 면** (좌·우·위) 에서 감싸 통제력 ↑.
- Intel 22 nm (2012, Tri-Gate) 첫 양산.
- TSMC 16 nm (2014), Samsung 14 nm (2015) 도입.

```
       Gate (3 면 감싸기)
         ┌─────┐
         │  ▓  │
       ──┤  ▓  ├──   ← Fin (vertical Si)
         │  ▓  │
         └─────┘
       substrate
```

다중 fin 묶음으로 더 강한 트랜지스터.

### GAAFET / Nanosheet (3 nm 이하)

- 채널을 가로 nanosheet 로, 게이트가 **4 면** 완전 감쌈 (Gate-All-Around).
- Samsung 3GAE (2022), TSMC N2 (2025+), Intel 18A.
- sheet 폭 조절로 transistor 강도 미세 조정.

```
       Gate (4 면)
         ┌─────┐
         │ ▓▓▓▓ │   ← Sheet 1
         │ ▓▓▓▓ │   ← Sheet 2 (수직 적층)
         │ ▓▓▓▓ │   ← Sheet 3
         └─────┘
```

### CFET (Complementary FET, 미래)

- nMOS·pMOS 를 수직 스택.
- 표준 셀 면적 절반.
- 1 nm 노드 이후 후보.

## 9. Strain / SOI / Stress Engineering

세대별 트랜지스터 강화:

### Strained Silicon (90 nm 부터)
- 실리콘 격자를 인장 (tensile) 또는 압축 (compressive) → 캐리어 이동도 ↑.
- nMOS = tensile (전자 이동도 ↑).
- pMOS = compressive (정공 이동도 ↑).
- SiGe 소스/드레인이 pMOS 의 압축 strain 제공.

### SOI (Silicon on Insulator)
- 절연막 (buried oxide) 위 얇은 실리콘 channel.
- substrate 와 절연 → junction 캐패시턴스 ↓ → 속도 ↑, 누설 ↓.
- IBM POWER (FD-SOI), Samsung 28FDS / 14FDS.
- Bulk vs FD-SOI 의 trade-off.

## 10. 회로 layout — Stick Diagram / DRC

### Stick Diagram (개념적 layout)

```
   Metal2  ████████████ Vdd
                  │
   Poly    ░░░░░░░░░░░    ← Gate (수직)
   ─pMOS─  ▓▓▓▓▓▓▓▓▓▓▓    ← p-active
   Metal1  ──────────     ← OUT
   ─nMOS─  ▓▓▓▓▓▓▓▓▓▓▓    ← n-active
   Poly    ░░░░░░░░░░░
   Metal2  ████████████ GND
```

표준 셀 layout 은 grid (track) 단위로 패턴화. 같은 height (track 수) 안에 모든 셀이 들어가도록 규격화.

### DRC (Design Rule Check)
- 공정 별 라이브러리에 minimum width / spacing / enclosure / via 위치 규칙.
- 위반 시 양산 불가.
- 자동 EDA tool (Synopsys, Cadence) 이 검증.

## 11. Body effect

- substrate 의 bias 가 V_th 를 끌어올리는 효과.
- 보통 source = body 로 연결되어 0 이지만, NMOS 가 직렬 stack 의 위 (source 가 GND 아닌) 면 body effect 발생.
- 회로 delay 분석에 영향.

## 12. Subthreshold Slope (SS)

- V_GS 가 V_th 아래에서 한 decade 의 I_D 변화당 V_GS 변화량.
- 단위: mV/decade.
- 이상 (Boltzmann limit): 60 mV/dec @ room temp.
- 실제 nm 공정: 65~90 mV/dec.

### 왜 중요
- ON/OFF ratio 가 SS 에 의해 결정.
- 작은 SS = 같은 V_dd 에서 더 큰 ON/OFF ratio = 누설 ↓.
- **양자 효과 / 새로운 트랜지스터 (TFET, MEMS)** 가 60 mV/dec 한계 깨려는 시도.

## 13. SPICE / 시뮬레이션

- **HSPICE / ngspice / Spectre** — 회로 시뮬레이터.
- BSIM4 / BSIM-CMG (FinFET) / BSIM-IMG (GAAFET) — 트랜지스터 모델.
- 한 게이트의 delay = ps 단위까지 정확히 시뮬.

```spice
* simple inverter
M1  out  in  vdd  vdd  pch  W=200n L=22n
M2  out  in  gnd  gnd  nch  W=100n L=22n
C1  out  0   1f
Vdd vdd  0   1
Vin in   0   pulse(0 1 0 50p 50p 1n 2n)
.tran 1p 5n
.end
```

## 14. ESD / Latch-up

### ESD (Electrostatic Discharge)
- 정전기로 트랜지스터 게이트 손상 (수 kV).
- 모든 IO pin 에 ESD diode (다이오드 + 큰 resistor).

### Latch-up
- CMOS 의 기생 (parasitic) bipolar 가 high current 로 self-sustaining 회로 형성.
- substrate 의 가까운 pMOS+nMOS 가 SCR 형태 → 칩 burn.
- guard ring + 적절한 spacing 으로 방지.

## 15. 함정

1. **V_th 가 고정이라는 가정** — 온도 / 공정 corner / 노화 (NBTI, HCI) 로 변동.
2. **누설 무시한 전력 모델** — 7 nm 이하부터 누설이 전력 절반.
3. **NAND 와 NOR 면적 동등 가정** — pMOS mobility ↓ 라 NOR 의 직렬 pMOS 가 더 큰 면적.
4. **클럭 빨리만 = 빠른 회로** — gate delay 가 짧아도 wire delay 와 fanout 이 천장.
5. **strain 효과 무시** — pMOS / nMOS 가 같은 W/L 이라도 ID 다름.
6. **fan-out 폭주** — 한 게이트 출력에 너무 많은 입력 매달면 slip → glitch.
7. **layout DRC 위반** — design rule 어기면 fab 가 reject. EDA tool 결과 신뢰.
8. **ESD 보호 미흡한 IO** — 사용자 단자에 trace 가 ESD diode 없이 노출.

## 16. 관련

- [[transistor]] — hub
- [[process-node-evolution]] — 공정 노드 / FinFET / GAAFET 진화의 산업적 의미
- [[../digital-logic/logic-gates]] — CMOS 게이트의 부울 추상화
- [[../digital-logic/sequential-circuits-clock]] — FF / register 가 트랜지스터로 구현되는 면
- [[../memory/dram-cell]] — DRAM 의 1T1C 셀
