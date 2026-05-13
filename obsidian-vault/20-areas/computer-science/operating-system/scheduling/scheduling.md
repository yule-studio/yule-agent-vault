---
title: "Scheduling (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:50:00+09:00
tags:
  - operating-system
  - scheduling
  - hub
---

# Scheduling (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 5 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

여러 ready 프로세스 / 스레드 중 **누구에게 CPU 를 줄 것인가** 결정.

목표:
- 공평 (fair)
- 처리량 (throughput)
- 응답 시간 (latency)
- 처리 시간 (turnaround)
- CPU 사용률

종종 trade-off — fair vs throughput vs latency.

---

## 2. 종류 (Preemption)

| 종류 | 의미 |
| --- | --- |
| **Non-preemptive** | task 가 자발 yield 까지 점유 |
| **Preemptive** | OS 가 timer 인터럽트로 강제 회수 |

현대 OS = preemptive (timer interrupt). 임베디드 / 협력적 멀티태스킹 = non-preemptive 잔존.

---

## 3. 알고리즘 한눈에

| 알고리즘 | 특징 | 노트 |
| --- | --- | --- |
| **FCFS** | 도착 순 | [[fcfs-sjf-rr]] |
| **SJF / SRTF** | 짧은 작업 우선 | [[fcfs-sjf-rr]] |
| **Round Robin** | timeslice | [[fcfs-sjf-rr]] |
| **Priority** | 우선순위 | (basic) |
| **MLFQ** | 다단계 피드백 | [[mlfq]] |
| **CFS** | Linux — Red-Black Tree, vruntime | [[cfs]] |
| **EDF** | Earliest Deadline First — 실시간 | [[realtime]] |
| **Rate Monotonic** | 짧은 주기 우선 — 실시간 | [[realtime]] |

---

## 4. Context Switch

자세히 → [[context-switch]]

스케줄러가 CPU 를 다른 task 에 넘길 때의 비용.

```
1. 현재 register / PC → task_struct
2. 새 task 의 register 복원
3. mm 다르면 TLB flush
4. cache cold
```

비용:
- 직접: ~1-10 μs
- 간접: 캐시 / TLB miss 로 수십 μs+

---

## 5. Linux 의 스케줄링

```c
struct sched_class {
    void (*enqueue_task)(...);
    void (*dequeue_task)(...);
    struct task_struct *(*pick_next_task)(...);
    // ...
};

// 우선순위 (높은 → 낮은)
stop_sched_class
dl_sched_class   (SCHED_DEADLINE — EDF)
rt_sched_class   (SCHED_FIFO / RR)
fair_sched_class (SCHED_OTHER — CFS)
idle_sched_class
```

대부분의 task = CFS. RT task 는 RT class 가 무조건 우선.

자세히 → [[cfs]], [[realtime]]

---

## 6. 정책 (Linux)

```bash
# 보기
chrt -p $PID

# 설정
nice -n 10 ./script
renice -n -5 -p $PID
chrt -f 50 ./realtime         # SCHED_FIFO, priority 50
chrt -r 50 ./realtime         # SCHED_RR
chrt -d --sched-runtime 30000 --sched-deadline 100000 --sched-period 100000 ./rt
```

| 정책 | 의미 |
| --- | --- |
| `SCHED_OTHER` (CFS, 기본) | 일반 |
| `SCHED_BATCH` | 비대화형 — 더 긴 timeslice |
| `SCHED_IDLE` | 가장 낮은 |
| `SCHED_FIFO` | 실시간, 양보 없음 |
| `SCHED_RR` | 실시간, RR |
| `SCHED_DEADLINE` | EDF 기반 |

---

## 7. nice / priority

| 정책 | 우선순위 |
| --- | --- |
| 일반 (CFS) | nice -20 ~ +19 (기본 0) |
| 실시간 | RT priority 1 ~ 99 |

- nice 작을수록 우선 (-20 = 최고)
- RT priority 높을수록 우선

```bash
ps -eo pid,ni,pri,policy,comm
# NI = nice, PRI = 내부 우선순위, POLICY = TS/FF/RR
```

---

## 8. CPU 친화도 (Affinity)

특정 task 를 특정 CPU 에 고정:

```bash
taskset -c 0,1 ./app
taskset -p 0x3 $PID         # mask
sched_setaffinity()
```

- 캐시 친화도 ↑
- NUMA 친화
- RT 워크로드 흔히 사용

---

## 9. Load Balancing (멀티코어)

CPU 마다 run queue. 부하 차이 → 다른 CPU 로 task 이동.

```bash
# 부하 확인
mpstat -P ALL 1
```

CFS 가 정기적으로 cross-CPU 부하 균형 — 옛 buddy CPU / 새 EAS (Energy Aware).

---

## 10. 인터럽트 vs Preemption

```
Timer interrupt → 스케줄러 깨움 → preempt 가능
인터럽트 핸들러 안 = 스케줄러 X
```

`CONFIG_PREEMPT_RT` (Linux RT) → 커널도 preemptible — 낮은 latency.

---

## 11. cgroups + CPU

```bash
echo 50000 > cgroup.cpu.max     # 50% quota (100000us 중 50000us)
echo $$ > cgroup.procs
```

자세히 → [[../virtualization/cgroups]]

---

## 12. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[fcfs-sjf-rr]] | 기본 알고리즘 |
| [[mlfq]] | 다단계 피드백 큐 |
| [[cfs]] | Linux Completely Fair Scheduler |
| [[realtime]] | EDF / Rate Monotonic / SCHED_FIFO/RR/DEADLINE |
| [[context-switch]] | 비용 / 측정 / 튜닝 |

---

## 13. 면접 핵심

1. **Preemptive vs Non-preemptive**.
2. **FCFS / SJF / RR** — 평균 대기 시간.
3. **MLFQ** 의 적응 — short / long task 자동 분류.
4. **CFS** — vruntime / Red-Black Tree.
5. **EDF / Rate Monotonic** — utilization bound.
6. **Context switch 비용**.
7. **Priority Inversion** — Mars Pathfinder.
8. **CPU affinity / NUMA**.
9. **Convoy effect** — FCFS 의 함정.
10. **Starvation** — Priority 의 함정.

---

## 14. 함정

### 14.1 RT priority 99 + 무한 loop
시스템 freeze. nice → RT 신중.

### 14.2 SCHED_FIFO + 양보 안 함
다른 task 영원 대기. sched_yield / sleep / RT throttling.

### 14.3 cgroup CPU 제한 oversubscription
throttle → tail latency 폭증.

### 14.4 nice = CPU 점유 보장 X
공평하게 줄여도 결국 share. quota 가 강제.

### 14.5 Affinity 잘못
스레드들이 1 코어에 몰림 → busy.

---

## 15. 학습 자료

- **OSTEP** Ch. 7-10
- **Linux Kernel Development** Ch. 4
- **The Linux Programming Interface** Ch. 35
- **brendangregg.com** — performance / scheduler

---

## 16. 관련

- [[../process/process]]
- [[../threads/threads]]
- [[context-switch]]
- [[../operating-system|↑ OS hub]]
