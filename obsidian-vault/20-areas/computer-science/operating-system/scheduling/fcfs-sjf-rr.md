---
title: "FCFS / SJF / RR — 기본 스케줄링 알고리즘"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:55:00+09:00
tags:
  - operating-system
  - scheduling
---

# FCFS / SJF / RR

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 4 가지 기본 |

**[[scheduling|↑ Scheduling hub]]**

---

## 1. 지표

| 지표 | 의미 |
| --- | --- |
| **Arrival Time** | 도착 시간 |
| **Burst Time** | 필요 CPU 시간 |
| **Completion Time** | 끝난 시간 |
| **Turnaround Time** | Completion − Arrival |
| **Waiting Time** | Turnaround − Burst |
| **Response Time** | 첫 실행까지 |

---

## 2. FCFS (First Come First Served)

도착 순. Non-preemptive.

```
P1 (24) | P2 (3) | P3 (3)
Time 0      24      27   30

P1 wait = 0, P2 = 24, P3 = 27
Avg waiting = (0+24+27)/3 = 17
```

### 2.1 Convoy Effect

긴 작업 뒤에 짧은 작업들이 모두 대기 → 평균 대기 시간 폭증.

→ 단순하지만 비효율.

---

## 3. SJF (Shortest Job First)

짧은 작업 먼저. Non-preemptive.

### 3.1 같은 예 (P2 (3), P3 (3), P1 (24))

```
P2 | P3 | P1
0    3    6      30

Avg waiting = (0+3+6)/3 = 3
```

→ FCFS 의 17 → 3.

### 3.2 SRTF (Shortest Remaining Time First) — Preemptive

새 task 도착 시 remaining time 짧으면 preempt.

### 3.3 단점
- 다음 burst time **모름** — 추정 (지수 가중 평균)
- 긴 작업 starvation

---

## 4. Priority Scheduling

각 task 우선순위. 높은 것 먼저.

```
Static — 처음에 정함
Dynamic — 시간에 따라 변함 (aging)
```

### 4.1 Starvation
낮은 우선순위가 영원히 대기.

### 4.2 Aging
대기 오래되면 우선순위 ↑.

---

## 5. Round Robin (RR)

각 task 에 **timeslice (quantum)** — 끝나면 ready queue 뒤로.

```
quantum = 4ms
P1 (24) P2 (3) P3 (3) RR (4)
```

```
P1 P2 P3 P1 P1 P1 P1 P1
0  4  7 10 14 18 22 26 30
```

→ 응답 시간 짧음 / fair.

### 5.1 quantum 선택

- 너무 작음 → context switch 비용 폭발
- 너무 큼 → FCFS 와 같아짐
- 보통 10-100 ms (대화형 OS) / 1-10 ms (서버)

Linux CFS quantum 은 적응적 (목표 latency / nr_running).

---

## 6. Multilevel Queue

여러 큐 + 큐 마다 다른 알고리즘:

```
Foreground (interactive) — RR
Background (batch) — FCFS
System (kernel) — Priority
```

- 큐 간 fixed priority 또는 timeshare
- task 큐 이동 안 함

→ MLFQ 가 진화형.

자세히 → [[mlfq]]

---

## 7. 비교 표

| 알고리즘 | Preempt | 평균 wait | 응답 | 구현 | 단점 |
| --- | --- | --- | --- | --- | --- |
| FCFS | ❌ | 큼 | 큼 | 단순 | Convoy |
| SJF | ❌ | 최적 | 가변 | 추정 필요 | starvation |
| SRTF | ✅ | 최적 | 좋음 | 복잡 | starvation |
| Priority | 가능 | 가변 | 가변 | 보통 | starvation |
| RR | ✅ | 보통 | 좋음 | 단순 | quantum 선택 |
| MLFQ | ✅ | 좋음 | 좋음 | 복잡 | 튜닝 |

---

## 8. 예제 계산

도착 P1 (0, 8), P2 (1, 4), P3 (2, 9), P4 (3, 5).

### 8.1 FCFS

```
P1 P2 P3 P4
0  8 12 21 26

wait = 0 + 7 + 10 + 18 = 35 / 4 = 8.75
```

### 8.2 SJF (non-preempt, after arrival)

P1 끝나면 P2(4), P3(9), P4(5) 중 P2(4) 먼저:

```
P1 P2 P4 P3
0  8 12 17 26

wait = 0 + 7 + 14 + 15 = 36 / 4 = 9
```

### 8.3 SRTF (preempt)

```
P1 P2 P2 P2 P2 P4 P1 P1 P1 P1 P1 P1 P1 P3 ...
T=0 doming Burst (CPU left) every step

result: P2 끝(5) → P4 끝(10) → P1 끝(17) → P3 끝(26)
wait = (17-0-8) + (5-1-4) + (26-2-9) + (10-3-5)
     = 9 + 0 + 15 + 2 = 26 / 4 = 6.5
```

→ SRTF 가 최적 평균 대기.

### 8.4 RR (q=4)

```
Time:    0  4  8 12 16 20 24
Run:    P1 P2 P3 P4 P3 P3
Queue: P2→P3→P4 ... P1 again? P1 다 끝 if 8 burst with q=4: 2 quanta
```

(상세 계산 생략 — RR 은 평균 wait 가 보통 SJF 보다 큼.)

---

## 9. 실무에서

거의 모든 현대 OS = preemptive + 우선순위 기반 + 다단계 (MLFQ / CFS).

| OS | 스케줄러 |
| --- | --- |
| Linux | CFS (SCHED_OTHER) + RT classes |
| Windows | Multilevel Feedback |
| macOS | Mach + BSD (priority) |
| FreeBSD | ULE |
| Solaris | 다단계 |

자세히 → [[cfs]]

---

## 10. 함정

### 10.1 FCFS 의 convoy
긴 batch 뒤의 짧은 interactive 가 latency ↑.

### 10.2 SJF 의 추정
미래 burst 정확히 모름. 지수 가중 평균.

### 10.3 RR quantum 작음
context switch 폭증.

### 10.4 Priority 의 starvation
aging.

### 10.5 학술 문제 vs 실무
실무는 거의 항상 hybrid. 학술은 비교 위한 단순화.

---

## 11. 학습 자료

- **OSTEP** Ch. 7
- **Modern Operating Systems** Ch. 2.4
- **Operating System Concepts** Ch. 6

---

## 12. 관련

- [[mlfq]] — 진화
- [[cfs]] — 현실 (Linux)
- [[scheduling]] — Scheduling hub
