---
title: "논리 게이트와 부울 대수"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, gate, boolean]
---

# 논리 게이트와 부울 대수

**[[digital-logic|↑ 디지털 논리]]**

## 1. 기본 게이트 진리표

| 게이트 | 식 | 00 | 01 | 10 | 11 |
| --- | --- | --- | --- | --- | --- |
| NOT | ¬A | 1 | — | 0 | — |
| AND | A·B | 0 | 0 | 0 | 1 |
| OR | A+B | 0 | 1 | 1 | 1 |
| XOR | A⊕B | 0 | 1 | 1 | 0 |
| NAND | ¬(A·B) | 1 | 1 | 1 | 0 |
| NOR | ¬(A+B) | 1 | 0 | 0 | 0 |
| XNOR | ¬(A⊕B) | 1 | 0 | 0 | 1 |

## 2. NAND / NOR universality

NAND 단독, 또는 NOR 단독으로 모든 부울 함수를 표현 가능.

- NOT = NAND(A,A).
- AND = NAND(NAND(A,B), NAND(A,B)).
- OR = NAND(NAND(A,A), NAND(B,B)).

→ fab 의 표준 셀 라이브러리는 NAND/NOR 위에 build.

## 3. 부울 대수 항등식

- **De Morgan**:
  - `¬(A·B) = ¬A + ¬B`
  - `¬(A+B) = ¬A · ¬B`
- **분배**: `A·(B+C) = A·B + A·C`.
- **흡수**: `A + A·B = A`, `A·(A+B) = A`.
- **consensus**: `A·B + ¬A·C + B·C = A·B + ¬A·C`.

## 4. Karnaugh Map (K-map)

논리 함수를 표 위에서 인접 1 묶음으로 그루핑 → 최소 곱항(SoP) / 합항(PoS).

| AB\CD | 00 | 01 | 11 | 10 |
| --- | --- | --- | --- | --- |
| 00 | 1 | 1 | 0 | 0 |
| 01 | 1 | 1 | 0 | 0 |
| 11 | 0 | 0 | 0 | 0 |
| 10 | 0 | 0 | 0 | 0 |

→ F = ¬A·¬C. (왼쪽 위 2x2 묶음)

4 변수까지 손으로, 그 이상은 Quine-McCluskey / ESPRESSO 같은 알고리즘.

## 5. 트랜지스터로 구현

CMOS NAND2 = pMOS 2 병렬 (pull-up) + nMOS 2 직렬 (pull-down). 게이트 개수가 적어 NOR 보다 nMOS 직렬 길이가 짧음 → 산업이 NAND 를 더 선호하는 이유 하나.

## 6. fan-in / fan-out / drive strength

- **fan-in** — 게이트 입력 수. 직렬 트랜지스터 수가 늘어 지연·면적 ↑.
- **fan-out** — 게이트 출력이 끌어야 하는 다음 입력 수. 캐패시턴스 ↑ → 지연 ↑.
- **drive strength** — 같은 NAND2 라도 X1 / X2 / X4 등 트랜지스터 폭이 다른 변종. 큰 X 가 빠르지만 면적·전력 ↑.

## 7. 함정

1. **De Morgan 잘못 적용** — 괄호 빠뜨림. `¬A + ¬B ≠ ¬(A + B)`.
2. **XOR 와 OR 혼동** — 자연어 "또는" 은 보통 inclusive (OR). 비트연산은 XOR/OR 명시.
3. **fan-out 폭주** — 한 출력에 너무 많이 매달면 신호 슬립.

## 8. 관련

- [[digital-logic]] — hub
- [[combinational-circuits]] — 게이트들로 만드는 다음 단계
- [[../transistor/mosfet-cmos]] — CMOS 게이트의 트랜지스터 구현
