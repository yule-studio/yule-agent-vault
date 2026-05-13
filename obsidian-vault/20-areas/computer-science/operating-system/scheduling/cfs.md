---
title: "CFS — Linux Completely Fair Scheduler"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:05:00+09:00
tags:
  - operating-system
  - scheduling
  - cfs
  - linux
---

# CFS — Completely Fair Scheduler

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CFS / vruntime / RB tree / group sched |

**[[scheduling|↑ Scheduling hub]]**

---

## 1. 한 줄

Linux 2.6.23+ 의 기본 스케줄러. 각 task 에게 **공평한 CPU 시간** 을 주려는 모델.
Ingo Molnár, 2007.

---

## 2. 핵심 아이디어 — vruntime

각 task 는 자기가 받은 CPU 시간을 **virtual runtime (vruntime)** 으로 누적.
스케줄러는 가장 작은 vruntime 의 task 를 선택.

```
모든 task 가 똑같이 vruntime 증가 → 모두가 같은 CPU 시간 받음
nice 가 다르면 가중치로 vruntime 증가율 조정
```

---

## 3. Red-Black Tree

ready 인 task 들을 vruntime 기준으로 RB tree 에 정렬.

```
pick_next_task = 최좌측 (leftmost) — O(log N)
새 task 삽입 = O(log N)
```

→ 수천 task 도 빠른 선택.

---

## 4. nice 와 weight

```
weight[nice] 표 (kernel/sched/core.c)

nice -20  → weight 88761
nice   0  → weight 1024
nice +19  → weight 15
```

vruntime 증가율:
```
delta_vruntime = delta_wall × (NICE_0_WEIGHT / weight)
```

nice -20 task = vruntime 천천히 증가 → 자주 선택 → 더 많은 CPU.

---

## 5. Target Latency / Min Granularity

```
sched_latency_ns          = 6,000,000 (기본 6ms)
sched_min_granularity_ns  =   750,000
sched_wakeup_granularity_ns = 1,000,000
```

- 모든 ready task 가 한 번씩 실행되는 목표 윈도우
- task 수 ≤ 8 → 6ms 안에 한 바퀴
- task 수 > 8 → 각 task 최소 750μs (min_granularity)

→ adaptive timeslice. RR 의 고정 quantum 과 다름.

---

## 6. Sleeper Fairness

I/O 로 잠들어 있던 task 가 깨어날 때 vruntime 너무 작아도 일정 보정.
→ 깨어난 후 다른 task 의 시간을 너무 빼앗지 않게.

---

## 7. 다중 CPU — Per-CPU Run Queue

```
CPU0: rq0 → RB tree
CPU1: rq1 → RB tree
...
```

주기적 load balance — 한 CPU 의 task 다른 CPU 로 마이그.

```
sched_domains
NUMA-aware
EAS (Energy Aware Scheduling, ARM)
```

---

## 8. Group Scheduling (cgroup)

```
사용자 / 그룹 단위 공평성
cgroup v2 cpu.weight (= sched weight)
```

```bash
# 두 그룹에 각각 100 task
# group A weight=200, group B weight=100
# → A 는 2/3, B 는 1/3 CPU
```

→ 한 사용자가 N task 띄워도 다른 사용자 CPU 빼앗지 못함.

---

## 9. 다른 sched class 와의 관계

```
stop_sched_class    (시스템 정지용 — migration)
dl_sched_class      SCHED_DEADLINE (EDF)
rt_sched_class      SCHED_FIFO / SCHED_RR (RT priority 1-99)
fair_sched_class    SCHED_OTHER (CFS) — 일반
idle_sched_class    idle
```

→ RT task 가 있으면 CFS 는 양보. RT task 가 다 끝나야 CFS 진행.

자세히 → [[realtime]]

---

## 10. CFS 의 한계

- **Latency 보장 X** — fair 하지만 실시간 데드라인 무관 → RT class.
- **NUMA / cache locality** — group scheduling / EAS 으로 점차 개선.
- **세분화된 latency 조절** — 모든 task 동등 → 응답성 차별화 어려움.

---

## 11. 측정 / 디버그

```bash
cat /proc/sched_debug
cat /proc/$PID/sched

# sysctl
cat /proc/sys/kernel/sched_latency_ns
cat /proc/sys/kernel/sched_min_granularity_ns

# 변경 (kernel.sched_* 는 일부 sysctl 비활성됨 5.10+)
echo 5000000 > /proc/sys/kernel/sched_latency_ns
```

```bash
# 컨텍스트 스위치 / migrate
pidstat -w 1
mpstat -P ALL 1
perf sched record / perf sched timehist
```

---

## 12. CFS bandwidth control (cgroup cpu.max)

```bash
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
# 100ms 윈도우 중 50ms 만 사용 — 50% quota
```

→ K8s CPU limit 의 토대. 자세히 → [[../virtualization/cgroups]]

⚠️ throttling = tail latency 폭증의 흔한 원인 (CPU 평균 적어도 burst 후 throttle).

---

## 13. EEVDF (5.6+, 6.6 default)

CFS 의 후계: **Earliest Eligible Virtual Deadline First**.
- vruntime + lag 기반
- 더 정확한 latency 모델
- nice 가 latency 에 영향

2023+ 표준. CFS 와 호환되는 인터페이스.

---

## 14. 함정

### 14.1 RT task + 무한 loop
일반 task 영원 starvation. RT throttling 기본 (sched_rt_runtime_us).

### 14.2 cgroup cpu.max 의 throttle spike
평균 CPU 적어도 burst 시 throttle → tail latency. quota 신중.

### 14.3 nice 가 CPU 비율 보장 X
다른 task 의 가중치에 상대적.

### 14.4 CFS 의 latency 보장 없음
RT class 사용.

### 14.5 group cpu.weight 만 사용
cpu.max 가 더 강한 강제.

### 14.6 NUMA 무시
cross-node 마이그 → 캐시 cold. taskset / numactl.

### 14.7 EEVDF 호환 가정
6.6+ 에서 일부 헷갈리는 동작 — kernel 버전 확인.

---

## 15. 학습 자료

- **Linux Kernel Development** Ch. 4
- **Documentation/scheduler/** (kernel source)
- **Brendan Gregg — scheduler perf**
- **LWN.net — CFS / EEVDF 시리즈**

---

## 16. 관련

- [[realtime]] — RT class
- [[context-switch]]
- [[../virtualization/cgroups]]
- [[scheduling]] — Scheduling hub
