---
title: "조합 회로 (Combinational Circuits)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, combinational, adder, mux, alu]
---

# 조합 회로 (Combinational Circuits)

**[[digital-logic|↑ 디지털 논리]]**

상태 없이 입력만으로 출력이 결정. 클록 신호 없음.

## 1. Adder

### Half Adder (1 bit, 캐리 입력 없음)
- `S = A ⊕ B`
- `C = A · B`

### Full Adder (1 bit, 캐리 입력 있음)
- `S = A ⊕ B ⊕ Cin`
- `Cout = A·B + Cin·(A ⊕ B)`

### n-bit Adder 방식

| 방식 | 면적 | 지연 |
| --- | --- | --- |
| **Ripple-Carry** | 작다 | O(n) — 캐리가 직렬 전파 |
| **Carry-Lookahead (CLA)** | 크다 | O(log n) |
| **Carry-Select** | 중 | O(√n) |
| **Kogge-Stone** | 매우 큼 | O(log n), tree 형태 |
| **Brent-Kung** | 중 | O(log n), 면적 절약 |

현대 CPU ALU 는 Kogge-Stone / Brent-Kung 변종. 더 빠른 가산기를 위해 면적·전력을 쓴다.

### Subtraction
2 의 보수로 처리: `A − B = A + ¬B + 1`. NOT 회로 + Adder + 입력 cin=1.

## 2. MUX (Multiplexer)

N 입력 중 1 개 선택해 출력.

```
2-to-1 MUX: Y = S·B + ¬S·A

   A ──┐
       │── ¬S
   B ──┤
       │── S
       └────► Y
   S ──┘
```

- 4-to-1, 8-to-1, … N-to-1 까지 확장.
- CPU 안에서 register file 읽기, 분기에 따른 데이터 선택 등 어디서나 사용.

## 3. Demux / Decoder

- **Demux** — MUX 의 반대. 1 입력을 N 출력 중 하나로 전달.
- **Decoder** — n bit → 2ⁿ 출력 중 정확히 1 개 활성. 메모리 row 선택, 인스트럭션 디코딩에 사용.

## 4. Encoder / Priority Encoder

- **Encoder** — 2ⁿ 입력 중 활성된 1 개의 인덱스를 n bit 로 출력.
- **Priority Encoder** — 여러 입력이 동시 활성 시 우선순위 가장 높은 입력의 인덱스 출력. **인터럽트 컨트롤러의 핵심**.

## 5. Comparator

- A == B, A < B, A > B 를 1 bit 신호로.
- 한 사이클 안에 모든 자리 동시 비교 (XNOR + AND 게이트 트리).

## 6. Barrel Shifter

임의 거리 shift / rotate 를 한 사이클에. MUX 트리로 구현.

- left shift, right shift, arithmetic right shift, rotate.
- SIMD (벡터 명령) / 암호 (rotate) / 직렬화에 필수.

## 7. ALU (Arithmetic Logic Unit)

조합 회로의 종합 — 산술 (+, -) + 논리 (AND/OR/XOR/NOT) + shift.

- 곱셈 / 나눗셈은 보통 별도 유닛 (지연 길어 multi-cycle).
- 1 사이클 ALU 가 CPU 클럭 주파수의 한계를 결정.

## 8. 함정

1. **glitch / hazard** — 입력 변화 도중 일시적 잘못된 출력. 클록 edge 안 쪽이면 무해, 비동기 출력으로 빠지면 위험.
2. **곱셈을 한 사이클에** — 32-bit 곱셈은 보통 3~5 사이클. 1 사이클 강요하면 클럭 천장 ↓.
3. **음수 처리 잊기** — `A − B` 가 부호 있는 비교에서 overflow 가능. signed 비교는 별도.

## 9. 관련

- [[digital-logic]]
- [[logic-gates]]
- [[sequential-circuits-clock]] — adder 뒤에 register 가 붙어 datapath 가 만들어진다
