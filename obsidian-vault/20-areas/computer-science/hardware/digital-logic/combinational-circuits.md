---
title: "조합 회로 (Combinational Circuits)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, combinational, adder, mux, decoder, alu, kogge-stone, barrel-shifter, multiplier]
---

# 조합 회로 (Combinational Circuits)

**[[digital-logic|↑ 디지털 논리]]**

> 상태 없이 입력만으로 출력 결정. CPU 의 datapath / ALU / 메모리 디코더 / 가속기의 PE 가 모두 조합 회로.

## 1. 조합 회로의 정의

- **출력 = 현재 입력의 함수만**. 과거 상태 의존 없음.
- 클럭 없음.
- 모든 게이트는 propagation delay (ps ~ ns) 동안 출력 결정.

### 순차 회로와 차이
- 조합: 즉시 (delay 후) 결정.
- 순차: 클럭 edge 에서 갱신, 상태 보유.

## 2. Adder — 가장 기초

### Half Adder (1 bit, 캐리 입력 없음)

```
inputs:  A, B
outputs: S (sum), C (carry)

S = A ⊕ B
C = A · B
```

```
    A ──┬──[XOR]──── S
        │      ╲
    B ──┼──────╳
        │      ╱
        └──[AND]──── C
```

### Full Adder (1 bit, 캐리 입력 있음)

```
inputs:  A, B, Cin
outputs: S, Cout

S = A ⊕ B ⊕ Cin
Cout = A·B + Cin·(A ⊕ B)
     = A·B + A·Cin + B·Cin   (대안)
```

```
   A ──┬──[XOR1]──┬──[XOR2]──── S
       │          │
   B ──┼──────────┘
       │      ╲
       │       ╳
       │      ╱
   Cin─┴──[AND]──┐
       │         │
   A·B ────[AND]─┤
                 │
                 [OR]─── Cout
```

### N-bit Adder — 방식 별 비교

| 방식 | 면적 | Delay |
| --- | --- | --- |
| **Ripple-Carry Adder (RCA)** | 작다 | O(n) — 캐리가 직렬 전파 |
| **Carry-Skip Adder** | 중 | O(√n) |
| **Carry-Select Adder** | 큼 | O(√n) — 두 결과 미리 + MUX |
| **Carry-Lookahead Adder (CLA)** | 큼 | O(log n) |
| **Kogge-Stone Adder** | 매우 큼 | O(log n) — 빠르지만 wire 많음 |
| **Brent-Kung Adder** | 중 | O(log n) — fewer wires than KS |
| **Sklansky Adder** | 중-큼 | O(log n) — 균형 |
| **Han-Carlson Adder** | 중 | O(log n) — BK + KS hybrid |

### Ripple-Carry — 단순하지만 느림

```
[FA0]──Cout──[FA1]──Cout──[FA2]──Cout──[FA3]
  │            │           │            │
S=A0+B0      S=A1+B1     S=A2+B2      S=A3+B3
```

- 캐리가 직렬 전파.
- 32-bit RCA = 32 delay = 매우 느림.

### Carry-Lookahead Adder (CLA)

캐리를 미리 병렬 계산.

**Generate / Propagate**:
- G_i = A_i · B_i (캐리 발생).
- P_i = A_i ⊕ B_i (캐리 전파).

```
Cout_i = G_i + P_i · Cin_i
       = G_i + P_i · (G_{i-1} + P_{i-1} · Cin_{i-1})
       ...
       = G_i + P_i·G_{i-1} + P_i·P_{i-1}·G_{i-2} + ...
```

→ 모든 캐리를 병렬 회로로 계산. O(log n) depth.

### Kogge-Stone Adder (KSA) — 현 CPU 표준

- prefix tree 구조.
- 모든 stage 의 input 이 다음 stage 의 모든 위치에 fan-out.
- 매우 빠름 (log n depth).
- wire 많음 (큰 면적, 큰 power).
- Intel / AMD / ARM 의 64-bit ALU 의 표준.

## 3. Subtractor

### 2 의 보수
- 가산기 + invert + Cin=1.
- `A − B = A + ¬B + 1`.

```
A ──────[Adder]── S
B ──[NOT]──┘  │
            Cin=1
```

## 4. Multiplier — 큰 회로

### Array Multiplier — N × N

```
       a3 a2 a1 a0
   ×   b3 b2 b1 b0
   ─────────────
       a3·b0 a2·b0 a1·b0 a0·b0
     a3·b1 a2·b1 a1·b1 a0·b1
   a3·b2 a2·b2 a1·b2 a0·b2
 a3·b3 a2·b3 a1·b3 a0·b3
 ────────────────────────
   sum of partial products
```

- 각 partial product = AND.
- 합 = adder array.

### Wallace Tree
- partial product 들을 tree 로 합성.
- log_2(n) depth.

### Booth Encoding
- 2 bit 한 번 처리 → partial product 수 절반.
- signed multiply 에 효율적.

### CPU 의 Multiplier
- 32×32 = 64 bit multiply.
- 보통 3-5 cycle latency.
- pipelined — 1 cycle throughput 가능.

## 5. MUX (Multiplexer)

N 입력 중 1 개 선택해 출력.

### 2-to-1 MUX

```
        A ──┐
            │
            │── if S=0
        ────┼────────► Y
            │
        B ──┘── if S=1

Y = S·B + ¬S·A
```

### N-to-1 MUX
- 4-to-1, 8-to-1, 16-to-1.
- log_2(N) select lines.
- tree of 2-to-1 MUX.

### MUX 의 용도
- CPU register file read port.
- 분기에 따른 데이터 선택.
- bus arbitration.

## 6. Demux / Decoder

### Demux
- 1 입력 → N 출력 중 하나 (others = 0).
- MUX 의 반대.

### Decoder
- n bit input → 2^n output (정확히 1 활성).
- 메모리 row 선택, instruction decoding 의 표준.

```
2-to-4 Decoder:
   A B │ Y0 Y1 Y2 Y3
   0 0 │  1  0  0  0
   0 1 │  0  1  0  0
   1 0 │  0  0  1  0
   1 1 │  0  0  0  1
```

### Tree decoder
- n bit decoder = n stage of 1-to-2 demux.
- 4096-row memory decoder = 12-stage decoder.

## 7. Encoder / Priority Encoder

### Encoder
- 2^n input 중 활성 한 개의 인덱스를 n bit 로 출력.
- 4-to-2:
  ```
   I0 I1 I2 I3 │ Y1 Y0
   1  0  0  0  │  0  0
   0  1  0  0  │  0  1
   0  0  1  0  │  1  0
   0  0  0  1  │  1  1
  ```

### Priority Encoder
- 여러 input 활성 시 우선순위 가장 높은 input 의 인덱스 출력.
- **인터럽트 컨트롤러의 핵심**.
- 예: IRQ 들 중 가장 priority 높은 거 처리.

### Variant
- One-hot encoder.
- Hot-bit search (CTZ — Count Trailing Zeros).
- BSF (Bit Scan Forward) instruction 의 구현.

## 8. Comparator

### Equal comparator
- A == B 를 1 bit 출력.
- XNOR + AND tree.

```
Equal = ¬(A0 ⊕ B0) · ¬(A1 ⊕ B1) · ... · ¬(An ⊕ Bn)
      = AND of (XNOR per bit)
```

### Magnitude comparator
- A < B, A == B, A > B 한 번에.
- carry chain (subtraction) 또는 dedicated tree.
- 한 사이클 안에 모든 자리 동시 비교.

## 9. Barrel Shifter

임의 거리 shift / rotate 를 **한 사이클**에.

### 기본 — N stage MUX tree

```
8-bit barrel shifter:
Stage 0: shift by 1
Stage 1: shift by 2
Stage 2: shift by 4

shift_amount 비트로 stage 활성 / 비활성.
shift = b2·4 + b1·2 + b0·1
```

### 종류
- **Left shift** — 왼쪽으로.
- **Right shift (logical)** — 오른쪽, 위 0 채움.
- **Right shift (arithmetic)** — 오른쪽, 위 MSB 채움 (signed).
- **Rotate** — 한쪽 끝에서 다른 쪽으로.

### 용도
- SIMD (vector 명령).
- 암호 (rotate 가 핵심 primitive).
- 직렬화 / packing.

## 10. ALU (Arithmetic Logic Unit)

조합 회로의 종합. 산술 + 논리 + shift.

### 기본 구성

```
A ──┬────────────────┐
    │                │
    │  ┌──[ADD]──┐   │
    │  │         │   │
    └──┤         │   │
       │  ┌─────────┐│
B ─────┤  │   MUX   ├┴── Y (output)
       │  │ ( op )  │
       │  └─────────┘
       │  ┌──[AND]──┐
       └──┤         │
          │  [OR]   │
          │  [XOR]  │
          │  [SHIFT]│
          └─────────┘
```

### 표준 연산
- ADD, SUB.
- AND, OR, XOR, NOT.
- Shift (left / right / arithmetic).
- Compare (==, <, >).
- 일부: BITWISE COUNT (POPCNT), LEADING ZERO.

### Cycle 별 처리
- 1 cycle ALU = ADD, SUB, AND/OR/XOR, shift.
- 별도 unit: multiply (3-5 cycle), divide (10-30 cycle), float (3-5).

### CPU 별 ALU 수
- Apple M1 P-core = 6 ALU (super-scalar).
- Intel Golden Cove = 5 ALU.
- AMD Zen 4 = 4 ALU.
- 하나의 cycle 에 다수 명령 동시 실행 가능.

## 11. Carry-Save Adder (CSA) — multi-input

### 동작
- 3 input → 2 output (sum + carry).
- 최종 합은 추가 adder 가 carry + sum 더함.
- multiplication / dot product 에 다수 사용.

```
Inputs: A, B, C
Outputs: S, Cout

S_i = A_i ⊕ B_i ⊕ C_i
Cout_i = majority(A_i, B_i, C_i) = A·B + B·C + A·C
```

### Wallace tree
- 다수의 CSA 를 tree 로 → 빠른 multi-input sum.

## 12. Look-up Table (LUT) — FPGA 의 기본

### FPGA 의 cell
- 4-input 또는 6-input LUT.
- 임의의 4 (or 6) 변수 함수를 구현.
- LUT + register + carry chain + MUX.

### LUT 구현
- 4-input LUT = 16-bit memory + 4-to-16 decoder + MUX.
- 모든 가능한 입력 조합의 출력 미리 저장.

### LUT 의 효용
- programmable — soft logic.
- 같은 silicon 으로 임의 회로 구성.
- ASIC 대비 10-100× 느림, 100-1000× 더 큰 면적.

## 13. ROM / Look-up Table — 조합 회로의 일반화

- 모든 조합 함수는 ROM 으로 표현 가능.
- N input function = 2^N row ROM.
- 큰 N 에는 비현실적 (2^32 row 면 4 GB).

→ K-map / ESPRESSO 로 단순화한 logic gate 가 일반적.

## 14. Iterative Combinational Circuit

같은 cell 을 반복.

### Bit-counter (popcount)

```
input: 32-bit X
output: 5-bit count of 1-bits

Method 1: linear adders.
Method 2: tree (log depth).
Method 3: SWAR (SIMD Within A Register) tricks.
```

### CRC (Cyclic Redundancy Check)
- iterative XOR + shift.
- Ethernet FCS, hash 등에 사용.
- HW 는 parallel CRC 회로.

## 15. 실 ALU 예 — RISC-V 32-bit

### Operations
- ADD, SUB, AND, OR, XOR, SLL (shift left), SRL, SRA.
- 추가: SLT (set less than), SLTU.

### 회로

```
   A ────────────┬────────────────────┐
                 │                    │
   B ────────────┼────────┐           │
                 │        │           │
   ┌──────────┐  │   ┌────▼────┐      │
   │  Adder/  ◄──┘   │ Subtra-  │     │
   │  Subtrct ├──────► ctor flag│     │
   └──────────┘                  │     │
                                 │     │
                              ┌──▼─────▼──┐
                              │   MUX     ├── Y
   AND ────────────────────►  │  (op)     │
   OR ─────────────────────►  │           │
   XOR ────────────────────►  │           │
   Barrel-Shift ───────────►  │           │
                              └───────────┘
```

## 16. Pipelined Combinational — 한 stage 안

순차 회로의 한 stage 안에 들어가는 조합 회로.

```
   FF ── Combinational Logic ── FF
   (input)   (1 사이클 안)   (output)
```

- combinational delay < 1 cycle period 여야 함.
- delay 가 너무 길면 pipeline 분할 필요.

## 17. 함정

1. **glitch / hazard** — 입력 변화 도중 일시적 잘못된 출력. 동기 회로 안이면 OK, 비동기 출력으로 빠지면 위험.
2. **곱셈을 한 사이클에** — 32-bit 곱셈은 보통 3-5 사이클. 1 사이클 강요하면 clock 천장 ↓.
3. **음수 처리 잊기** — `A − B` 의 overflow. signed / unsigned 비교 별도.
4. **CLA / KSA 의 큰 wire** — 큰 면적 / power. 32-bit 까지는 OK, 64-bit 에서 신중.
5. **MUX 의 select decoding 부담** — 큰 MUX 의 select 가 늘 critical path.
6. **bit-shift 의 sign extension** — arithmetic right shift vs logical right shift 구분.
7. **CSA 의 final adder 빠뜨림** — sum + carry 가 최종 결과가 아님.
8. **LUT 가 RAM 처럼 사용 가능** 오해 — FPGA LUT 는 static. 동적 update 별도 mechanism (LUTRAM).

## 18. 관련

- [[digital-logic]]
- [[logic-gates]] — 게이트 위 구성
- [[sequential-circuits-clock]] — adder 뒤에 register 가 붙어 datapath 가 만들어진다
- [[../transistor/mosfet-cmos]] — 게이트 / 회로의 트랜지스터 단
- [[../../computer-architecture/computer-architecture]] — CPU 의 datapath / ALU / FPU
