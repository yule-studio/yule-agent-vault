---
title: "MLFQ — Multilevel Feedback Queue"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:00:00+09:00
tags:
  - operating-system
  - scheduling
  - mlfq
---

# MLFQ — Multilevel Feedback Queue

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | MLFQ 5 규칙 |

**[[scheduling|↑ Scheduling hub]]**

---

## 1. 한 줄

미리 burst time 모르지만 **관찰로 학습**:
- 짧게 끝남 = interactive → 우선순위 ↑
- 길게 점유 = batch → 우선순위 ↓

→ SJF 의 정신 + RR 의 공평성. CTSS (1962) 의 원형.

---

## 2. 기본 규칙 (Arpaci-Dusseau)

### 규칙 1
우선순위 (A) > 우선순위 (B) → A 실행.

### 규칙 2
같은 우선순위 → RR.

### 규칙 3
새 작업 = **최고 우선순위** (낙관).

### 규칙 4
한 큐의 timeslice 모두 쓰면 우선순위 ↓ (CPU bound 라 판단).

### 규칙 5
**Priority Boost** — 주기적으로 모든 task 를 최고 우선순위로 (starvation 방지 + 모드 변화).

---

## 3. 동작 시각화

```
Priority 8: [interactive task]      (받자마자)
Priority 7: [I/O bound, 짧은 burst]
Priority 6: ...
...
Priority 1: [CPU bound batch]

CPU 가 짧은 burst 끝나면 yield → 같은 큐 유지
timeslice 다 씀 → 한 단계 ↓
```

---

## 4. 적응의 핵심

- I/O 자주 = 짧은 burst → 우선순위 유지
- 무한 loop = 긴 burst → 점차 낮아짐
- batch ↔ interactive 모드 변화도 booster 가 잡음

→ **사용자 입력 / GUI** 가 자연스럽게 빠른 응답.

---

## 5. Gaming the System

task 가 의도적으로 timeslice 직전 yield → 우선순위 유지.

→ **누적 timeslice** 으로 측정 (한 큐에서 누적 = 일정 후 강등).

---

## 6. Boost 주기

너무 짧음 → CPU bound 가 너무 자주 부활 → batch 도 응답 OK 이지만 fairness ↑
너무 김 → starvation 가능

보통 1-10 초.

---

## 7. Linux 의 옛 — O(1) Scheduler

2.6.0 ~ 2.6.22 (2003-2007) 사이 사용.

```
140 priority levels (0~99 RT, 100~139 일반)
각 priority 마다 큐
active / expired array
"interactivity" heuristic 으로 boost
→ O(1) pick_next_task
```

문제:
- heuristic 복잡 / 휴리스틱 의존 / corner case
- → **CFS** 로 교체 (2.6.23)

---

## 8. Solaris / macOS / Windows

각자의 MLFQ 변형. 현재도 사용:

### 8.1 Windows
32 priority. Interactive 작업 boost, foreground 가산점.

### 8.2 macOS
Mach + BSD. 4 band (timeshare, interactive, system, RT).

### 8.3 FreeBSD ULE
Round-robin + priority adjustment.

---

## 9. MLFQ 의 한계

- heuristic 복잡 / 튜닝
- gaming
- 공평성 명시 X (CFS 가 해결)
- 응답성 vs throughput 의 명시적 trade-off X

→ CFS 가 더 단순한 모델로 해결.

자세히 → [[cfs]]

---

## 10. MLFQ vs CFS

| | MLFQ | CFS |
| --- | --- | --- |
| 모델 | 큐 + 휴리스틱 | virtual runtime |
| pick | O(1) 또는 O(N) | O(log N) RB tree |
| 공평성 | 명시 X | 명시 (vruntime) |
| 응답성 | boost 로 | 자동 (적은 vruntime 이 짧은 task) |

이론적으로 CFS 가 더 깨끗. 그러나 RT / batch 분리는 여전히 별도 정책.

---

## 11. 함정

### 11.1 Heuristic 튜닝의 함정
워크로드 따라 다름 → 모든 경우 잘 되는 값 없음.

### 11.2 Gaming
의도적 yield → 강등 회피. 누적 timeslice 측정.

### 11.3 RT class 분리 누락
긴 RT task = MLFQ 안에서 처리 X. 별도 class.

### 11.4 Priority Boost 주기
시스템 다양한 워크로드에 맞춰 측정 필요.

---

## 12. 학습 자료

- **OSTEP** Ch. 8 (MLFQ)
- **Linux Kernel Development** Ch. 4 (구 O(1))
- **Solaris Internals** — 옛

---

## 13. 관련

- [[fcfs-sjf-rr]] — 기본 알고리즘
- [[cfs]] — 진화
- [[scheduling]] — Scheduling hub
