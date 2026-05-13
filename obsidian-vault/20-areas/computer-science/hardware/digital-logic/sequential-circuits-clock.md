---
title: "순차 회로와 클록 (Sequential Circuits / Clock)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, flip-flop, clock, fsm, cdc, setup, hold, skew, jitter, pll, metastability, pipeline]
---

# 순차 회로와 클록

**[[digital-logic|↑ 디지털 논리]]**

> 상태가 있고 클록에 동기. 시간이 흐르는 모든 디지털 회로 — CPU register, 캐시, FSM, pipeline.

## 1. 메모리 셀의 최소 단위

| 셀 | 트리거 | 1 bit 저장 | 비고 |
| --- | --- | --- | --- |
| **SR Latch** | level (S/R 신호) | ✓ | set / reset 라인. 메타스테이블 위험. |
| **D Latch** | level (enable) | ✓ | enable=1 동안 D 통과 (transparent). |
| **D Flip-Flop (DFF)** | edge (rising/falling) | ✓ | 클록 edge 에서 D 캡처. 동기 회로 기본. |
| **JK Flip-Flop** | edge | ✓ | toggle / counter 에 사용. |
| **T Flip-Flop** | edge | ✓ | toggle (input 1 → flip). |
| **Master-Slave FF** | edge | ✓ | 두 latch 직렬, edge-triggered 의 구현. |

## 2. SR Latch — 가장 기초

### NOR-based SR Latch

```
      S ──┐
          │
        [NOR]─── Q
          │
        [NOR]─── Q̄
          │
      R ──┘
```

### 진리표

| S | R | Q | Q̄ | 상태 |
| --- | --- | --- | --- | --- |
| 0 | 0 | (유지) | (유지) | hold |
| 1 | 0 | 1 | 0 | set |
| 0 | 1 | 0 | 1 | reset |
| 1 | 1 | 0 | 0 | **invalid** (Q = Q̄) |

### 문제
- S=R=1 후 동시 0 으로 떨어지면 Q 가 0/1 어느 안정점에 갈지 race.
- → SR Latch 그대로 사용 안 함. D latch / D FF 로.

## 3. D Latch — Transparent Latch

```
      D ─────┐
             │
       en ──[gate]── S
             │
             │
       en ──[gate]── R
             │
      ¬D ────┘
                │
            [SR Latch]
                │
                Q
```

- en = 1 일 때 D 가 Q 로 그대로 통과.
- en = 0 일 때 Q 가 마지막 값 유지.

### Transparent (level-triggered)
- en 이 HIGH 동안 D 변화가 Q 에 그대로 반영.
- 동기 회로의 한 stage 로 부적합 (race 위험).

## 4. D Flip-Flop (DFF) — 표준 셀

### Edge-triggered
- 클록의 rising edge (또는 falling edge) **순간**에만 D 캡처.
- 그 외 시간 = D 변경해도 Q 안 변함.

### Master-Slave 구현

```
     Master Latch (en=¬clk)
                │
     ────────── │ ──── Slave Latch (en=clk)
     D          Q'                 │
                                   Q
```

- clk=0: Master transparent (D → Q'). Slave hold.
- clk=1: Master hold (Q' fixed). Slave transparent (Q' → Q).
- → clk 의 rising edge 에서 D → Q 한 번만.

### CMOS Implementation
- Master + Slave 각 4-6 transistor.
- 한 DFF = 20-30 transistor.

### Async reset / set
- 일부 DFF 는 reset / set 핀 (clock 무관 즉시 0/1).
- 시스템 power-on 후 초기화.

## 5. Register / Counter / Shift Register

### Register
- DFF n 개 묶음.
- n bit 값 1 사이클 저장.
- CPU 의 GPR (General Purpose Register), PC, IR 등이 모두 register.

### Counter

#### Ripple counter (비동기)
- 각 FF 의 clock 이 이전 FF 의 Q.
- 단순. delay 가 stage 마다 누적 → 느림.

#### Synchronous counter
- 모든 FF 가 같은 clock.
- 더 빠름, 더 예측 가능.

#### Johnson counter
- ring + invert.
- 2N 상태 (N FF 로).

#### LFSR (Linear Feedback Shift Register)
- XOR feedback.
- pseudo-random sequence 생성.
- CRC, scrambler, 암호 stream.

### Shift Register
- DFF 직렬.
- 매 cycle 한 bit shift.
- SPI / UART / JTAG 통신의 핵심 부품.
- parallel ↔ serial 변환.

## 6. FSM (Finite State Machine)

### 기본
- **상태 (State)** + **입력 (Input)** → **다음 상태 + 출력**.

### Mealy vs Moore

| 종류 | 출력 결정 | 특징 |
| --- | --- | --- |
| **Mealy** | 입력 + 상태 | 빠름 (현 입력 즉시 반영). glitch 위험. |
| **Moore** | 상태만 | 안정 (glitch 적음). 출력 1 cycle 지연. |

### Mealy 회로 구조

```
   inputs ──┬─────────────────────────────┐
            │                             │
            │  ┌──────────────────────┐   │
            │  │ next-state logic     │   │
            └─►│ (combinational)      │   │
               └─────────┬────────────┘   │
                         │                │
                      ┌──▼──┐              │
                      │ FF  │ ← state      │
                      └──┬──┘              │
                         │                  │
                         ├──────────────────┤
                         │                  │
                   ┌─────▼─────┐            │
                   │ output    │◄───────────┘
                   │ logic     │
                   │ (Mealy:   │
                   │  inputs+  │
                   │  state)   │
                   └─────┬─────┘
                         │
                       output
```

### Moore 회로
- output 이 state 만 의존.
- output logic 이 inputs 안 받음.

### FSM 예 — Traffic Light

```
States: GREEN, YELLOW, RED.
Input: timer expired.

GREEN  ── timer ──► YELLOW
YELLOW ── timer ──► RED
RED    ── timer ──► GREEN
```

→ 3 state = 2 FF (binary encoding). 또는 one-hot encoding = 3 FF.

### One-hot vs binary encoding
- **Binary**: log_2(N) FF. 작음, decoding 필요.
- **One-hot**: N FF. decoding 0, 더 빠름.
- FPGA = one-hot 선호 (LUT 풍부).
- ASIC = binary 일반.

## 7. 클록 — 박자

### 클록 신호
- square wave, duty cycle 50% (보통).
- 모든 동기 FF 의 박자.
- 1 GHz = 10⁹ cycles/sec → period 1 ns.

### Clock distribution
- 모든 FF 에 균등하게 도착해야 함.
- 큰 chip 의 clock = millions of FF 까지 분배.

#### H-tree distribution
```
        Source
          │
       ───┼───
       │     │
     ──┼──  ──┼──
     │   │  │   │
     ┼  ┼  ┼  ┼   ← H-tree 의 leaf 끝마다 같은 거리
```

- 모든 leaf 까지 등거리.
- 한 leaf 에 doesn't reach 시간 동일.

#### Mesh
- H-tree 위 mesh 한 겹.
- skew 더 작음.
- 큰 chip (CPU 의 die) 에서 흔함.

#### Multi-clock domain
- 한 chip 안 여러 clock (CPU clock, memory clock, IO clock, display clock).
- 도메인 사이 신호 전달 = CDC (Clock Domain Crossing) 회로 필수.

## 8. 핵심 타이밍 — Setup / Hold / Skew / Jitter

### Setup Time (t_su)
- 클록 edge **전** D 가 안정해야 하는 시간.
- typical 100-200 ps.

### Hold Time (t_h)
- 클록 edge **후** D 가 유지되어야 하는 시간.
- typical 50-100 ps.

### Setup / Hold 위반 시 — Metastability

- D 가 setup window 내에 변하면 FF 의 internal state 가 0/1 가운데 빠짐.
- 일정 시간 후 무작위로 0 또는 1 로 안정.
- **다음 FF 가 이 값을 받으면 무작위 버그**.

### Setup 식

```
t_clk-period ≥ t_clk-q + t_logic + t_setup + t_skew
```

- t_clk-q: FF 의 clock → Q 지연.
- t_logic: 조합 회로 지연.
- t_setup: 다음 FF 의 setup.
- t_skew: 두 FF 사이 clock 도착 시간 차.

### Hold 식

```
t_clk-q + t_logic ≥ t_hold + t_skew (대상 FF)
```

- 너무 빠른 path 가 hold 위반.
- 작은 logic depth 의 short path 에서 문제.

### Skew

- 두 FF 사이 clock 도착 시간 차.
- **Positive skew (downstream 가 늦게)** → setup 더 여유, hold 위험.
- **Negative skew (downstream 가 빨리)** → setup 위험, hold 여유.

### Jitter
- 클록 edge 의 시간축 흔들림 (period 별 분산).
- typical 1-10 ps RMS.
- PLL / PHY 의 핵심 지표.

## 9. PLL / DLL — 클록 생성

### PLL (Phase-Locked Loop)
- 외부 reference (예: 25 MHz crystal) 를 받아 곱해서 GHz clock 만듦.
- VCO (Voltage-Controlled Oscillator) + phase detector + filter.
- 핵심: jitter 최소화.

```
   Reference ── phase detector ── filter ── VCO ── output
                       ▲                      │
                       └──── feedback ────────┘
                                 ÷ N (divider)
```

### DLL (Delay-Locked Loop)
- 곱셈 안 함. 위상만 맞춤.
- 더 단순, jitter 더 적음.

### 사용
- CPU clock generation.
- DDR memory clock recovery.
- PCIe / USB SerDes.
- Display clock.

## 10. Multi-clock Domain — CDC

### 문제
- 두 clock domain 사이 신호 직결 시 setup / hold 보장 불가.
- 메타스테이블 → 무작위 버그.

### Synchronizer — 2-stage FF

```
   source domain      destination domain
                 │             │
   src_signal ──┤── FF1 ── FF2 ── safe signal
                 │
              clk_src       clk_dst
```

- FF1 에 메타스테이블 일어나도 FF2 에서 안정.
- MTBF (Mean Time Between Failure) 계산 가능 — typical 10^6 + 년.

### Handshake — 다중 bit 전달

```
   src                           dst
   data ──────────────────────► data
   req ──[sync]──► ack ──[sync]──┘
```

- req / ack 1-bit 신호 동기.
- data 는 stable 보장.

### Async FIFO — 다중 bit 큰 throughput

```
   src write port      dst read port
        │                   │
        write_ptr      read_ptr
        │                   │
        Memory (dual-port)
```

- Gray code pointer 사용 (한 비트만 변경 보장).
- 양 domain 의 pointer 가 안전하게 비교 가능.

## 11. Reset — 시스템 시작

### 동기 vs 비동기 reset

#### Asynchronous reset
- clock 무관 즉시 reset.
- 응급 / power-on.
- 단점: release 시 메타스테이블 가능.

#### Synchronous reset
- clock edge 에서만 reset.
- 안전.
- 단점: clock 이 동작해야 reset 가능.

### Best practice
- async assert + sync deassert.
- power-on 시 async (clock 없음).
- 정상 동작 시 sync.

## 12. Pipeline — 조합 회로 분할

### 개념
- 긴 조합 회로를 stage 로 분할.
- 각 stage 사이 register 삽입.
- 클럭 ↑, throughput ↑, latency 같음 또는 ↑.

### 5-stage RISC pipeline

```
   IF → ID → EX → MEM → WB
   │    │    │    │     │
   FF   FF   FF   FF    FF
```

각 stage 가 1 cycle.

### Stall / Bubble
- data dependency 또는 branch 로 한 stage 멈춤.
- 다음 stage 에 NOP (bubble) 삽입.

### Hazard
- **Data hazard**: 앞 instruction 결과 의존.
- **Control hazard**: branch / jump.
- **Structural hazard**: 자원 충돌.

### 해법
- **Forwarding** — ALU output 을 다음 stage input 으로 직접.
- **Branch prediction** — 추측 + speculative execution.
- **Reorder buffer** — out-of-order 실행.

자세히는 [[../../computer-architecture/computer-architecture]].

## 13. STA (Static Timing Analysis)

### 무엇
- 시뮬레이션 없이 회로의 모든 path 의 timing 계산.
- 모든 setup / hold violation 검출.

### 도구
- Synopsys PrimeTime — 산업 표준.
- Cadence Tempus.

### 결과
- **Slack** = 여유 시간. positive = OK, negative = violation.
- **Critical path** = slack 최소인 path.
- chip 의 max clock = 1 / critical path delay.

### Sign-off
- chip tape-out 전 모든 corner (process, voltage, temperature) 에서 STA pass.
- TSMC / Samsung 등 fab 이 timing library 제공.

## 14. Power Gating / Clock Gating

### Clock Gating
- 사용 안 하는 block 의 clock 차단.
- ICG (Integrated Clock Gate) cell.
- 동적 전력 ↓ (P = α·C·V²·f 에서 α → 0).

### Power Gating
- 사용 안 하는 block 의 전원 차단.
- header / footer transistor.
- 누설 power → 0.

### 단점
- 전원 켜기 시간 (power-up sequence).
- voltage glitching 위험.

## 15. HDL — Verilog / SystemVerilog / VHDL / Chisel

### Verilog — DFF 예

```verilog
module dff(
  input  clk, rst, d,
  output reg q
);
  always @(posedge clk or posedge rst) begin
    if (rst)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

### Counter

```verilog
module counter(
  input clk, rst,
  output reg [3:0] count
);
  always @(posedge clk) begin
    if (rst)
      count <= 4'b0;
    else
      count <= count + 1;
  end
endmodule
```

### FSM (Moore, traffic light)

```verilog
module traffic(
  input clk, rst,
  input timer_expired,
  output reg [1:0] state
);
  parameter GREEN = 2'b00, YELLOW = 2'b01, RED = 2'b10;

  always @(posedge clk) begin
    if (rst)
      state <= GREEN;
    else case (state)
      GREEN:  if (timer_expired) state <= YELLOW;
      YELLOW: if (timer_expired) state <= RED;
      RED:    if (timer_expired) state <= GREEN;
    endcase
  end
endmodule
```

## 16. Asynchronous Logic — 클록 없는 회로

### 동기 vs 비동기
- 동기: 모든 변화가 clock 에 맞춰.
- 비동기: 신호 도착 즉시 다음 단계.

### 장점
- 클록 분배 부담 없음.
- 평균 case 빠름.
- low-power (사용 안 할 때 0 power).

### 단점
- 설계 매우 어려움 — race, hazard 디버깅.
- EDA tool 의 support 적음.

### 사용
- 작은 임베디드 (Cortex-M0+).
- 일부 academic / research chip.
- 산업에서 매우 드묾.

## 17. 함정

1. **클록 게이팅 잘못** — clock 라인에 AND gate 직접 → glitch. 전용 ICG cell 사용.
2. **두 클록 도메인 직결** — 메타스테이블 → 무작위 버그. 반드시 synchronizer.
3. **setup violation 무시** — 시뮬레이션에서 안 잡히고 실 칩에서만 발생. STA 가 잡아냄.
4. **글로벌 reset 비동기 풀림** — 모두 동시에 풀리면 FF 들이 메타스테이블. synchronous reset release.
5. **hold 위반 무시** — setup 보다 fix 어려움 (buffer 추가 또는 path 재설계).
6. **multi-clock 같은 sensitivity list** — Verilog 의 `always @(posedge clk1 or posedge clk2)` 은 합성 못 함.
7. **DFF 의 X 전파** — 시뮬레이션에서 X (unknown) 가 전파 → buggy 동작 가림. 명시적 reset.
8. **블로킹 vs 논블로킹** — Verilog 의 `=` (blocking) vs `<=` (non-blocking). FF 는 `<=` 만.
9. **One-hot encoding 의 illegal state** — 일부 FF stuck 시 illegal state. always default 처리.
10. **clock 의 negedge / posedge 혼용** — 같은 design 안 두 종 사용은 setup/hold 분석 복잡.
11. **PLL lock time 무시** — startup 시 PLL 가 안정될 때까지 50-100 μs.
12. **scan chain (DFT) 추가 후 timing** — test mode 의 timing 영향.

## 18. 관련

- [[digital-logic]]
- [[logic-gates]] — 게이트 위 단계
- [[combinational-circuits]] — pipeline stage 안 내용
- [[../transistor/mosfet-cmos]] — FF 의 트랜지스터 구현
- [[../../computer-architecture/computer-architecture]] — CPU pipeline / hazard / forwarding
