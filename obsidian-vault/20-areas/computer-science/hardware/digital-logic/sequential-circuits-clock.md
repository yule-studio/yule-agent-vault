---
title: "순차 회로와 클록 (Sequential Circuits / Clock)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, digital-logic, flip-flop, clock, fsm, cdc]
---

# 순차 회로와 클록

**[[digital-logic|↑ 디지털 논리]]**

상태가 있고 클록에 동기. 시간이 흐르는 모든 디지털 회로.

## 1. 메모리 셀 단위 — Latch / Flip-Flop

| 셀 | 트리거 | 비고 |
| --- | --- | --- |
| **SR Latch** | level | set/reset. 메타스테이블 위험. |
| **D Latch** | level (enable) | enable=1 동안 D 통과. transparent. |
| **D Flip-Flop (DFF)** | edge (rising/falling) | 클록 edge 에서 D 캡처. 모든 동기 회로 기본 셀. |
| **JK / T Flip-Flop** | edge | counters 등에 사용. |
| **Master-Slave FF** | edge | 두 latch 직렬, 클록 양 phase 활용. |

## 2. Register / Counter / Shift Register

- **Register** — DFF n 개 묶음. n bit 값 1 사이클 저장.
- **Counter** — 1 사이클마다 +1. ripple counter (비동기) / synchronous counter (동기) / Johnson counter / LFSR (pseudo-random).
- **Shift Register** — 직렬↔병렬 변환. SPI / UART / JTAG 통신의 핵심 부품.

## 3. FSM (Finite State Machine)

상태 + 입력 → 다음 상태 + 출력.

| 종류 | 출력 결정 | 응답 |
| --- | --- | --- |
| **Mealy** | 입력 + 상태 | 빠름 (현 입력 즉시 반영) |
| **Moore** | 상태만 | 안정 (glitch 적음) |

```
        ┌─────► next-state logic ──────┐
        │                              │
   inputs                              ▼
        │                            ┌──┐
        │                            │FF│  state
        │                            └──┘
        │                              │
        └─────► output logic ◄─────────┘
                  (Mealy 는 inputs 도 들어옴)
```

## 4. 클록 — 박자, skew, jitter

- 모든 동기 회로의 박자. GHz = 10⁹ cycles/sec.
- **clock period** = 1/freq. 3 GHz → 333 ps.
- 한 사이클 안에서 ① FF 출력 ② 조합 회로 통과 ③ 다음 FF 의 setup 까지 모두 끝나야 함.

### 타이밍 제약

- **setup time (t_su)** — 클록 edge 전 D 가 안정해야 하는 시간.
- **hold time (t_h)** — 클록 edge 후 D 가 유지되어야 하는 시간.

`t_period ≥ t_clk-q + t_logic + t_setup + t_skew`

- **t_clk-q** — FF 의 clock→Q 지연.
- **t_logic** — 조합 회로 지연.
- **t_skew** — 두 FF 사이 클록 도착 시간 차.

setup 위반 시 **메타스테이블** — 출력이 0 도 1 도 아닌 중간값에서 임의 시간 후 결정. 그 결과를 다음 FF 가 받으면 무작위 버그.

### Jitter / skew 줄이는 기법

- **H-tree clock distribution** — 클록을 H 모양으로 분배, 모든 leaf 까지 거의 같은 거리.
- **Clock mesh** — H-tree 위에 mesh 한 겹.
- **PLL (Phase-Locked Loop)** / **DLL** — 외부 reference 와 위상 맞춤 + 주파수 곱셈.

## 5. Clock Domain Crossing (CDC)

칩 한 개에 여러 클록 도메인 (CPU / 메모리 / IO / 디스플레이). 도메인 사이 신호는 반드시 **CDC 회로** 로:

- **2-stage synchronizer** — 2 개 DFF 직렬, 첫 번째에서 메타스테이블 일어나도 두 번째에서 안정될 확률 ↑.
- **handshake (req/ack)** — 양 도메인 신호로 한 word 전달.
- **async FIFO** — 양 도메인 사이 깊이 ≥2 의 FIFO. gray code pointer 사용.

## 6. Pipeline

조합 회로를 stage 로 쪼개 register 를 사이에 삽입.

- 결과: 한 사이클에 끝나야 했던 긴 path 가 여러 사이클로 분할 → 클럭 ↑.
- 비용: latency ↑ (한 명령 끝나는 시간), hazard (data / control / structural) 처리 로직.

CPU 의 5 단 / 14 단 / 20 단 파이프라인이 이 패턴. 자세히는 [[../../computer-architecture/computer-architecture]].

## 7. 함정

1. **클록 게이팅 잘못** — clock 라인에 AND 게이트 직접 → glitch. 전용 ICG (Integrated Clock Gate) cell 사용.
2. **두 클록 도메인 직결** — 메타스테이블 → 무작위 버그. 반드시 synchronizer.
3. **setup violation 무시** — 시뮬레이션에서 안 잡히고 실 칩에서만 발생. STA (Static Timing Analysis) 가 잡아냄.
4. **글로벌 reset 비동기 풀림** — 모두 동시에 풀리면 FF 들이 메타스테이블. **synchronous reset release** 권장.

## 8. 관련

- [[digital-logic]]
- [[combinational-circuits]] — pipeline 의 한 stage 안 내용
- [[../transistor/mosfet-cmos]] — FF 가 트랜지스터로 구현되는 단
