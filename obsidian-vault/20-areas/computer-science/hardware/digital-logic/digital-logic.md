---
title: "디지털 논리 (Digital Logic) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, gate, flip-flop, clock]
---

# 디지털 논리 (Digital Logic) — Hub

**[[../hardware|↑ hardware]]**

> 전자가 흘러서 0/1 을 의미하게 만드는 가장 아래 층. CPU·메모리·SoC 가 모두 이 층 위에 서 있다.

---

## 0. 한 줄

**부울 대수 + 클록 + 트랜지스터 라이브러리로 임의의 디지털 회로를 표현하는 분야.**

## 1. 비트 / 바이트 / 워드

| 단위 | 크기 | 의미 |
| --- | --- | --- |
| bit | 1 | 0 또는 1 |
| nibble | 4 bit | 16 진수 한 자리 |
| byte | 8 bit | 메모리 주소 최소 단위 |
| word | 32/64 bit | CPU 처리 단위 |

### 수의 표현

- **부호 없는 정수** — `uint64`: 0 ~ 18446744073709551615.
- **2 의 보수 (signed)** — `int64`: -9.22e18 ~ 9.22e18. MSB 가 부호.
- **IEEE 754 binary32 (float)** — 1 sign / 8 exp / 23 mantissa, ≈ 7 digit 정밀도.
- **IEEE 754 binary64 (double)** — 1 / 11 / 52, ≈ 15 digit.
- **bfloat16** — 1 / 8 / 7. float32 와 exponent 동일 → ML 친화.
- **FP8 (E4M3 / E5M2)** — 1 / 4or5 / 3or2. NVIDIA Hopper 이후 가속기 표준.

## 2. 논리 게이트 / 부울 대수
→ [[logic-gates|논리 게이트와 부울 대수]]

NAND·NOR 각자만으로 모든 회로 표현 가능 (universal). 그래서 fab 은 NAND 표준 셀 위에 라이브러리를 짜고 그 위에 AND/OR/XOR/MUX 등을 합성.

## 3. 조합 회로
→ [[combinational-circuits|조합 회로]]

상태 없이 입력만으로 출력이 결정되는 회로. Adder / MUX / Decoder / Encoder / Priority Encoder / Barrel Shifter / ALU.

## 4. 순차 회로 + 클록
→ [[sequential-circuits-clock|순차 회로와 클록]]

상태가 있고 클록 edge 에 동기. Latch / Flip-Flop / Register / Counter / FSM. Setup/hold/skew/jitter 의 의미.

## 5. HDL — Verilog / SystemVerilog / VHDL / Chisel

```verilog
module adder32(
  input  [31:0] a, b,
  input         cin,
  output [31:0] sum,
  output        cout
);
  assign {cout, sum} = a + b + cin;
endmodule
```

- Verilog/SystemVerilog — 산업 표준.
- VHDL — Ada 풍, EU·정부 권에 많이.
- Chisel / SpinalHDL — Scala DSL, RISC-V 진영.

## 6. 함정

1. **메타스테이블** — setup/hold 위반 시 출력이 임의값 / 지연. 두 클록 도메인 사이는 반드시 synchronizer.
2. **클록 skew** — H-tree / mesh 분배. 0 에 가깝게.
3. **유한 fan-out** — 1 개 게이트 출력이 너무 많은 입력을 끌면 지연.

## 7. 관련

- [[../transistor/transistor]] — 게이트가 트랜지스터로 구현되는 다음 단
- [[../memory/memory]] — SRAM/DRAM 셀
- [[../../computer-architecture/computer-architecture]] — CPU 단
