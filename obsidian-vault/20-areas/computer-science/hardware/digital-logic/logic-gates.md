---
title: "논리 게이트와 부울 대수"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, gate, boolean, de-morgan, nand, kmap, cmos, universality]
---

# 논리 게이트와 부울 대수

**[[digital-logic|↑ 디지털 논리]]**

> 모든 디지털 회로의 가장 작은 추상화 단위. CMOS NAND 한 게이트에서 시작해 CPU 한 개까지.

## 1. 기본 게이트 — 진리표

### 7 개 기본 게이트

| 게이트 | 식 | 00 | 01 | 10 | 11 |
| --- | --- | --- | --- | --- | --- |
| **NOT (Inverter)** | ¬A | 1 → 0, 0 → 1 |
| **AND** | A·B | 0 | 0 | 0 | 1 |
| **OR** | A+B | 0 | 1 | 1 | 1 |
| **XOR (Exclusive OR)** | A⊕B | 0 | 1 | 1 | 0 |
| **NAND (NOT AND)** | ¬(A·B) | 1 | 1 | 1 | 0 |
| **NOR (NOT OR)** | ¬(A+B) | 1 | 0 | 0 | 0 |
| **XNOR (NOT XOR, equivalence)** | ¬(A⊕B) | 1 | 0 | 0 | 1 |

### 기호 (IEEE / IEC)

| 게이트 | IEEE 모양 |
| --- | --- |
| NOT | 삼각형 + 작은 원 (bubble) |
| AND | D 모양 (flat back) |
| OR | 굽은 back |
| XOR | OR + 추가 곡선 |
| NAND | AND + bubble |
| NOR | OR + bubble |

## 2. NAND / NOR Universality

### 정리
**NAND 단독**, **또는 NOR 단독**으로 모든 부울 함수를 표현 가능.

### NAND 로 모든 게이트 구성

```
NOT(A)      = NAND(A, A)

AND(A, B)   = NAND(NAND(A, B), NAND(A, B))
            = NOT(NAND(A, B))

OR(A, B)    = NAND(NAND(A, A), NAND(B, B))
            = NAND(NOT A, NOT B)
            = NOT((NOT A)·(NOT B))         ← De Morgan
            = A + B  ✓

XOR(A, B)   = NAND(NAND(A, NAND(A, B)),
                   NAND(B, NAND(A, B)))    ← 4 NAND
```

### Fab 의 영향
- 표준 셀 라이브러리의 기본 셀 = NAND, NOR.
- 다른 게이트는 NAND/NOR 위에 build.
- 1 종 cell 만 검증하면 layer 위로 확장 가능 → 양산 단순화.

### CMOS NAND vs NOR — 면적 차이

- NAND2 = pMOS 2 병렬 + nMOS 2 직렬.
- NOR2 = pMOS 2 직렬 + nMOS 2 병렬.
- pMOS 의 mobility (정공) < nMOS (전자) ≈ 1/2.5.
- → 같은 strength 위한 pMOS 가 더 큼.
- NOR 의 pMOS 직렬 = 2 개 모두 큼.
- NAND 가 더 작고 빠르다.
- → **NAND 가 산업의 기본 셀**.

자세히는 [[../transistor/mosfet-cmos]] §6.

## 3. 부울 대수 항등식

### 기본 항등식

| 항등식 | 식 |
| --- | --- |
| Identity | A + 0 = A, A · 1 = A |
| Null | A + 1 = 1, A · 0 = 0 |
| Idempotent | A + A = A, A · A = A |
| Inverse (complement) | A + ¬A = 1, A · ¬A = 0 |
| Double negation | ¬¬A = A |
| Commutative | A + B = B + A, A · B = B · A |
| Associative | (A + B) + C = A + (B + C) |
| Distributive | A · (B + C) = A·B + A·C |

### De Morgan 의 법칙

가장 중요. 디지털 회로 분석 / 단순화의 핵심.

```
¬(A · B) = ¬A + ¬B
¬(A + B) = ¬A · ¬B
```

### Absorption

```
A + A·B = A
A · (A + B) = A
A + ¬A·B = A + B
A · (¬A + B) = A·B
```

### Consensus

```
A·B + ¬A·C + B·C = A·B + ¬A·C
(A + B)·(¬A + C)·(B + C) = (A + B)·(¬A + C)
```

마지막 항이 중간이라 빠짐 — 회로 단순화에 사용.

### XOR identities

```
A ⊕ 0 = A
A ⊕ 1 = ¬A
A ⊕ A = 0
A ⊕ ¬A = 1
A ⊕ B = ¬(A ⊕ ¬B)
A ⊕ B = (A · ¬B) + (¬A · B)
```

## 4. 표현 형태 — SoP vs PoS

### Sum of Products (SoP)
- AND term 들의 OR.
- 예: `F = A·B + ¬A·C + B·¬C`.
- "minterm" — 모든 변수 포함된 product.

### Product of Sums (PoS)
- OR term 들의 AND.
- 예: `F = (A + B)·(¬A + C)·(B + ¬C)`.
- "maxterm" — 모든 변수 포함된 sum.

### 전환
- 진리표에서 1 인 row → minterm 의 OR (SoP).
- 진리표에서 0 인 row → maxterm 의 AND (PoS).

## 5. Karnaugh Map (K-map) — 시각적 단순화

논리 함수를 2D 표 위에서 인접 1 묶음으로 그루핑 → 최소 SoP / PoS.

### 2 변수
```
     B=0  B=1
A=0   X    Y
A=1   Z    W
```

### 3 변수 (Gray code)
```
        BC=00  01  11  10
A=0      .    .   .   .
A=1      .    .   .   .
```

### 4 변수
```
          CD=00  01  11  10
AB=00      .    .   .   .
AB=01      .    .   .   .
AB=11      .    .   .   .
AB=10      .    .   .   .
```

### 예제

```
   AB\CD  00  01  11  10
   00      1   1   0   0
   01      1   1   0   0
   11      0   0   0   0
   10      0   0   0   0
```

→ 왼쪽 위 2×2 묶음 → F = ¬A · ¬C.

### 묶음 규칙
1. 1 의 인접한 그룹을 큰 사각형으로 (1, 2, 4, 8, 16 개).
2. 가장 큰 사각형 우선.
3. 모든 1 이 최소 한 사각형에 포함.
4. 인접 = 가장자리 wrap 도 포함 (Gray code).

### Don't Care
- 어떤 input 조합이 절대 일어나지 않는다고 알면 X 로 표시.
- 묶음에 X 를 1 처럼 포함시켜 더 큰 사각형 만들 수 있음.

### K-map 의 한계
- 4 변수까지 손으로 가능.
- 5-6 변수도 가능하지만 복잡.
- 6+ 변수 = **Quine-McCluskey** 알고리즘 또는 **ESPRESSO** (UC Berkeley) 자동.

## 6. ESPRESSO / Quine-McCluskey

### Quine-McCluskey
- 1950s 의 알고리즘.
- prime implicant 찾기 + minimum cover.
- exact (optimal) result.
- 큰 input 에 시간 폭증.

### ESPRESSO
- 1980s UC Berkeley.
- heuristic — exact 가 아니지만 매우 빠름.
- 거의 모든 EDA tool (Synopsys, Cadence) 의 logic minimization 표준.

## 7. CMOS 로 게이트 구현

### CMOS Inverter (NOT)

```
       Vdd
        │
      ┌─┴─┐
      │pMOS│   ← gate = A
      └─┬─┘
        │── Y = ¬A
      ┌─┴─┐
      │nMOS│   ← gate = A
      └─┬─┘
        │
       GND
```

- A=0 → pMOS ON, nMOS OFF → Y = Vdd = 1.
- A=1 → pMOS OFF, nMOS ON → Y = GND = 0.

### CMOS NAND2

```
            Vdd
        ┌────┴─────┐
       pA│        │pB     pMOS 2 병렬
        ┴          ┴
        ────┬──┬────
            │  │
            └──┘── Y = ¬(A·B)
            ┌──┐
        ────┴──┴────
        ┬          ┬
       nA│        │nB     nMOS 2 직렬
        └────┬─────┘
            GND
```

- A=B=1 → pMOS 둘 다 OFF, nMOS 둘 다 ON → Y = GND = 0.
- 그 외 → Y = Vdd = 1.

→ 트랜지스터 4 개.

### CMOS NOR2

```
            Vdd
        ┌────┴─────┐
       pA│        pA       pMOS 2 직렬
        ┴
        ────┬───────
            │
       pB   │ pB
            │
        ────┬──── Y = ¬(A+B)
            ┌──┐ ┌──┐
        ────┴──┴─┴──┴─
        ┬        ┬
       nA       nB     nMOS 2 병렬
        └────┬───┴
            GND
```

- pMOS 직렬은 mobility ↓ 라 더 큰 트랜지스터.

### CMOS AND / OR — 별도 cell 또는 NAND + Inverter

```
   AND2 = NAND2 → Inverter   (트랜지스터 6 개)
   OR2  = NOR2  → Inverter   (트랜지스터 6 개)
```

→ AND / OR 가 NAND / NOR 보다 큰 이유.

## 8. Logical Effort — 게이트 delay 분석

### 식
```
d = g · h + p
```

- **d**: stage 의 delay (시간 단위).
- **g**: logical effort (게이트 종류 별 상수).
- **h**: electrical effort (load capacitance / input capacitance).
- **p**: parasitic delay (게이트 자체 caps).

### Logical effort 표

| 게이트 | g |
| --- | --- |
| Inverter | 1 |
| NAND2 | 4/3 |
| NOR2 | 5/3 |
| NAND3 | 5/3 |
| NOR3 | 7/3 |
| XOR2 | 4 |
| MUX2 | 2 |

→ **fan-in 클수록 g 큼** (직렬 트랜지스터의 누적).

### Path 최적화
- N stage 의 path 가 있을 때, 각 stage 의 effort 가 같으면 path 의 total delay 최소.
- "balanced effort" — buffer 삽입 / size 조정.

## 9. Fan-in / Fan-out / Drive strength

### Fan-in
- 게이트 입력 수.
- 직렬 트랜지스터 수가 늘어 delay ↑, 면적 ↑.
- 4 입력 이상은 보통 2-stage 로 split (NAND2 + Inverter chain).

### Fan-out
- 게이트 출력이 끌어야 하는 다음 입력 수.
- 각 다음 입력 = gate capacitance.
- 더 많은 fan-out = 더 큰 load = 더 긴 delay.
- 일반 limit: 4-8 (그 이상은 buffer chain).

### Drive strength (X1, X2, X4, ...)

- 같은 NAND2 라도 트랜지스터 width 다른 변종.
- 큰 X = 강한 drive (빠름) + 큰 면적 + 큰 power.

```
NAND2_X1 → 가벼운 fan-out 1-2 용
NAND2_X4 → 중간 fan-out 5-10 용
NAND2_X16 → 큰 bus driver
```

합성 (synthesis) 도구가 path timing 요구에 맞춰 자동 선택.

## 10. Glitch / Hazard

### Static hazard
- 입력 1 비트만 변하는데 출력이 일시적 0 → 1 → 0 (또는 1 → 0 → 1).
- 신호 propagation 시간 차이로.
- 동기 회로 (clock edge 안 끝남) 면 무해.
- 비동기 출력으로 빠지면 위험.

### Dynamic hazard
- 여러 비트 동시 변경 시 multi-glitch.

### 대응
- redundant cover 추가 (K-map 의 모든 prime implicant 포함).
- 동기화 (synchronizer).

## 11. Tri-state / Open-drain — 특수 출력

### Tri-state buffer
- 출력 = 0 / 1 / **Z (high impedance)**.
- bus 의 여러 driver 가 share 하는 환경.
- "enable" 신호로 Z 전환.

### Open-drain / Open-collector
- 출력 = 0 또는 floating (Z).
- 외부 pull-up resistor 가 1 로 끌어줌.
- I²C / 1-Wire / SDA / SCL 의 표준.
- "wired-AND" 가능 — 모든 driver 가 1 일 때만 line 이 1.

## 12. CMOS 의 noise margin

### V_OH / V_OL / V_IH / V_IL
- V_OH (output high) = output 1 의 최소 전압. typical 0.9 × Vdd.
- V_OL (output low) = output 0 의 최대 전압. typical 0.1 × Vdd.
- V_IH (input high) = input 이 1 로 해석되는 최소.
- V_IL (input low) = input 이 0 으로 해석되는 최대.

### Noise margin
- NM_H = V_OH - V_IH (1 의 margin).
- NM_L = V_IL - V_OL (0 의 margin).
- 둘 다 같은 게 ideal (V_dd/2 transition).

### CMOS 의 강점
- regenerative — 약간 noisy signal 도 깨끗한 0 / 1 로 복원.
- noise margin 크다 (Vdd/2 정도).

## 13. Logic family — 가족

| Family | 특징 | 시대 |
| --- | --- | --- |
| **RTL (Resistor-Transistor)** | 1960s, 느림 | IBM System/360 |
| **DTL (Diode-Transistor)** | RTL 보다 빠름 | |
| **TTL (Transistor-Transistor)** | 1970-80s, 5V | 74-series chip |
| **ECL (Emitter-Coupled)** | 매우 빠름, 전력 ↑ | 슈퍼컴 |
| **CMOS** | 1970s+, 전력 ↓ | 현 표준 |
| **BiCMOS** | bipolar + CMOS hybrid | 90s 일부 |

현 디지털 회로의 99% = CMOS.

## 14. 진리표 → 회로 변환 (Logic synthesis)

### 단계
1. 진리표 작성.
2. SoP 또는 PoS 형태로.
3. K-map 또는 ESPRESSO 로 단순화.
4. 표준 셀 (NAND/NOR/MUX) 로 매핑.
5. Timing closure (delay 계산 + size 조정).

### EDA 도구
- **Synopsys Design Compiler** — 산업 표준.
- **Cadence Genus** — 동급.
- **Mentor / Siemens Catapult** — high-level synthesis.
- **Yosys** — 오픈소스.

## 15. Sample 회로 — Full Adder

### 1-bit Full Adder

```
inputs:  A, B, Cin
outputs: S (sum), Cout
```

### 진리표

```
A B Cin | S Cout
0 0 0   | 0  0
0 0 1   | 1  0
0 1 0   | 1  0
0 1 1   | 0  1
1 0 0   | 1  0
1 0 1   | 0  1
1 1 0   | 0  1
1 1 1   | 1  1
```

### SoP

```
S = A·¬B·¬Cin + ¬A·B·¬Cin + ¬A·¬B·Cin + A·B·Cin
  = A ⊕ B ⊕ Cin

Cout = A·B + Cin·(A ⊕ B)
     = A·B + A·Cin + B·Cin   (대안 형태)
```

### 게이트 구성

```
S    = XOR(XOR(A, B), Cin)              (2 XOR)
Cout = OR(AND(A, B), AND(Cin, XOR(A, B))) (1 OR, 2 AND, 1 XOR — XOR sharing)
```

→ 트랜지스터 ~30-40 개 / full adder.

## 16. 함정

1. **De Morgan 잘못 적용** — 괄호 빠뜨림. `¬A + ¬B ≠ ¬(A + B)`.
2. **XOR 와 OR 혼동** — 자연어 "또는" 은 보통 inclusive (OR). 비트연산은 XOR/OR 명시.
3. **fan-out 폭주** — 한 출력에 너무 많이 매달면 신호 슬립 / glitch.
4. **K-map 의 wrap-around 잊기** — 4 변수 K-map 의 모서리도 인접.
5. **logic synthesis 의 결과 신뢰** — EDA tool 의 단순화는 timing-driven. 면적 / power 우선 별도 옵션.
6. **CMOS 의 정적 누설 0 가정** — 7 nm 이하부터 누설이 동적 power 와 비슷.
7. **Tri-state bus 의 contention** — 동시에 두 driver enable → short circuit.
8. **Open-drain 의 pull-up 없음** — line 이 항상 floating. external resistor 필수.
9. **Glitch 가 동기 회로에서도 문제** 가정 — clock edge 안 끝나면 무해.
10. **logical effort 의 wire delay 무시** — 작은 회로는 OK. 큰 chip 에서는 wire 가 주.

## 17. 관련

- [[digital-logic]]
- [[combinational-circuits]] — 게이트들로 만드는 다음 단계
- [[sequential-circuits-clock]] — 게이트 + FF 결합
- [[../transistor/mosfet-cmos]] — CMOS 게이트의 트랜지스터 구현
