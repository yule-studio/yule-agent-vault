---
title: "실시간 스케줄링 — EDF / RM / SCHED_FIFO / SCHED_RR / SCHED_DEADLINE"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:10:00+09:00
tags:
  - operating-system
  - scheduling
  - realtime
---

# 실시간 스케줄링

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | EDF / RM / Linux RT |

**[[scheduling|↑ Scheduling hub]]**

---

## 1. Real-time 의 정의

**deadline 안에 끝나야 한다**.

| 종류 | 의미 | 예 |
| --- | --- | --- |
| Hard RT | deadline 미스 = 시스템 실패 | 비행기 제어 / 자동차 ECU |
| Firm RT | 미스 시 결과 무가치 | 비디오 frame drop |
| Soft RT | 미스 시 가치 ↓ | 음성 통화 / 게임 |

Linux 의 RT 는 보통 **soft / firm**. Hard RT 는 RTOS (FreeRTOS, VxWorks, QNX).

---

## 2. 두 가지 표준 모델

### 2.1 Rate Monotonic (Liu & Layland 1973)

- **주기 짧을수록 우선순위 높음**.
- 정적 우선순위.
- Utilization 조건:

```
U = Σ Cᵢ / Tᵢ ≤ n × (2^(1/n) - 1)
n=1 → 1.00
n=2 → 0.83
n=∞ → 0.69
```

- Cᵢ = 실행 시간, Tᵢ = 주기.

### 2.2 EDF (Earliest Deadline First)

- **가장 빠른 deadline 우선**.
- 동적 우선순위.
- Utilization 조건:

```
U = Σ Cᵢ / Tᵢ ≤ 1
```

→ 이론적으로 EDF 가 우월. 단, 과부하 시 cascade 실패 (preemption 폭증).

---

## 3. Linux RT 정책

```c
SCHED_FIFO     // 우선순위 + 양보 없음 (자기 정지/yield 전까지)
SCHED_RR       // 우선순위 + 같은 priority 끼리 RR
SCHED_DEADLINE // EDF (deadline 기반)
```

| | priority | 양보 |
| --- | --- | --- |
| FIFO | 1-99 | sched_yield, block 만 |
| RR | 1-99 | timeslice (sched_rr_timeslice_ms) |
| DEADLINE | 자체 | runtime, deadline, period |

CFS 위에 항상 우선. 즉 RT task = 일반 task starve.

---

## 4. 설정

```bash
chrt -f 50 ./mybin           # SCHED_FIFO priority 50
chrt -r 50 ./mybin           # SCHED_RR

chrt --deadline \
    --sched-runtime 30000   \
    --sched-deadline 100000 \
    --sched-period 100000   \
    ./mybin
# C=30ms, D=100ms, T=100ms
```

```c
struct sched_attr attr = {
    .size = sizeof(attr),
    .sched_policy = SCHED_DEADLINE,
    .sched_runtime  = 30000000,
    .sched_deadline = 100000000,
    .sched_period   = 100000000,
};
syscall(SYS_sched_setattr, 0, &attr, 0);
```

---

## 5. RT throttling

```bash
cat /proc/sys/kernel/sched_rt_period_us         # 1000000 (1초)
cat /proc/sys/kernel/sched_rt_runtime_us         #  950000 (0.95초)
```

→ 1 초 중 0.95 초만 RT 사용 → 일반 task 가 최소 50ms 보장.
무한 loop RT task 가 시스템 freeze 시키지 않게.

비활성:
```bash
echo -1 > /proc/sys/kernel/sched_rt_runtime_us
```

⚠️ 잘못된 RT task = 시스템 dead. 신중.

---

## 6. CPU 자원 보호

```bash
# RT task 만 사용할 CPU 격리
isolcpus=2,3              # boot param
nohz_full=2,3
rcu_nocbs=2,3

taskset -c 2,3 ./rt
```

특정 코어를 OS 일반 부하에서 격리 → 더 일관된 latency.

---

## 7. PREEMPT_RT 패치 (mainline 통합 중, 6.12+)

- 커널 자체가 fully preemptible
- spinlock → mutex (preemptible)
- 인터럽트 스레드화
- → microsecond 단위 latency

전통적 RTOS 에 가까운 Linux. 자동차 / IoT / industry.

---

## 8. Priority Inversion + Inheritance

```
Low priority L 이 mutex 잡음
High priority H 가 mutex 대기
Medium M 이 L preempt → H 영원 대기
```

해결:
- **Priority Inheritance** — L 의 우선순위를 H 까지 boost
- **Priority Ceiling** — mutex 자체 ceiling 설정

```c
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
```

자세히 → [[../synchronization/mutex#7-priority-inversion]]

Mars Pathfinder 1997 — Priority Inversion 사고. VxWorks 의 PIP off → 강제 reboot.

---

## 9. CPU 친화 + RT

```bash
chrt -f 50 taskset -c 2 ./rt_task
```

같은 CPU pin → 캐시 + TLB 일관성 + 다른 CPU 의 일반 부하 영향 X.

---

## 10. Latency 측정

```bash
# cyclictest (rt-tests)
sudo cyclictest -p 80 -t 1 -n -i 1000

# 결과
# T: 0 (1234) P:80 I:1000 C: 10000 Min: 5 Act: 8 Avg: 7 Max: 23
```

P80 priority, interval 1000μs, jitter 측정.

---

## 11. RTOS

| RTOS | 특징 |
| --- | --- |
| **FreeRTOS** | 임베디드 / IoT (오픈) |
| **Zephyr** | Linux Foundation, IoT |
| **VxWorks** | Wind River, 우주 / 항공 |
| **QNX** | BlackBerry, 자동차 |
| **NuttX** | UAV, 임베디드 |
| **RT-Linux / Xenomai** | Linux + hard RT |

→ 진짜 hard RT 는 RTOS. Linux 는 soft RT 까지 (PREEMPT_RT 로 좁힘).

---

## 12. 컨테이너 / K8s 의 RT

CPU pinning + dedicated cgroup + PREEMPT_RT 결합:
- 5G RAN
- High-frequency trading
- 산업 제어

→ 일반 K8s 는 latency 보장 X. RT 환경은 별도 설계.

---

## 13. 함정

### 13.1 RT 무한 loop → freeze
sched_rt_runtime_us 가 보호 — 그래도 nice 0 task 가 50ms / 1s 만 받음.

### 13.2 SCHED_FIFO 가 sleep 안 함
spin / busy → 다른 모든 task starve. sched_yield + 짧은 sleep.

### 13.3 RT 가 mutex 잡음
Priority Inversion. PIP mutex.

### 13.4 GC 가 있는 언어 + RT
Java GC pause = ms 단위. ZGC / Shenandoah 검토 또는 native.

### 13.5 Page fault
RT task 가 swap 으로 떨어짐 = ms 폭증. `mlockall(MCL_CURRENT|MCL_FUTURE)`.

### 13.6 cgroup CPU limit + RT
quota 도달 시 throttle. RT 와 충돌.

### 13.7 Hyperthread / Frequency scaling
ms 단위 jitter. HT off + governor=performance.

---

## 14. 학습 자료

- **Real-Time Systems** — Jane Liu
- **Linux RT documentation** — wiki.linuxfoundation.org/realtime
- **PREEMPT_RT** — 커널 패치 / 통합 노트
- **cyclictest / rt-tests**

---

## 15. 관련

- [[cfs]] — 일반 task
- [[../synchronization/mutex]] — Priority Inheritance
- [[scheduling]] — Scheduling hub
