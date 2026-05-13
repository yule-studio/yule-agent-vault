---
title: "Context Switch — 컨텍스트 스위치 비용"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:15:00+09:00
tags:
  - operating-system
  - scheduling
  - context-switch
---

# Context Switch

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 비용 / 측정 / 튜닝 |

**[[scheduling|↑ Scheduling hub]]**

---

## 1. 한 줄

CPU 가 한 task → 다른 task 로 전환하는 동작. **직접 + 간접** 비용 모두 큼.

---

## 2. 동작 — 무엇이 바뀌나

```
1. 현재 task 의 register / PC → task_struct
2. 다음 task 의 register 복원
3. mm 다르면:
   - CR3 변경 (페이지 테이블 루트 교체)
   - TLB flush (PCID 없으면)
4. 가상 메모리의 캐시 페이지 hot/cold 가 변함
5. instruction pipeline flush
```

→ 명령 자체는 빠름. **간접 비용 (cache, TLB miss)** 가 큼.

---

## 3. 비용

| 종류 | 비용 |
| --- | --- |
| 같은 process 의 thread 스위치 | ~1-3 μs (mm 같음, TLB 유지) |
| 다른 process 스위치 | ~5-10 μs (TLB flush) |
| 간접 (cache cold) | 수십 μs ~ ms |

→ 1 ms 가 CPU 의 수백만 cycle.

---

## 4. 빈도 측정

```bash
vmstat 1
# cs 컬럼 = context switch / 초

pidstat -w 1
# 프로세스별 cswch (voluntary) + nvcswch (non-voluntary)

perf stat -e cs,migrations sleep 10
```

높은 cs:
- timeslice 만료 (CPU bound, RR)
- 자주 block (I/O bound)
- 락 경합

---

## 5. Voluntary vs Involuntary

| | Voluntary | Involuntary |
| --- | --- | --- |
| 원인 | block (I/O, lock) | timeslice 만료, preempt |
| 일반 모드 | sleep / wait | preempt by scheduler |
| 의미 | I/O bound | CPU bound + 경쟁 |

---

## 6. Same Process Thread Switch

```
mm 같음 → CR3 변경 X → TLB 유지
스레드 stack / register 만 교체
```

→ 프로세스 스위치보다 훨씬 가벼움.

→ 멀티스레드 의 한 이유.

---

## 7. PCID (Process Context ID) — x86-64

```
CR3 의 일부에 ASID/PCID
같은 mm 의 page 들이 TLB 에 남음
→ context switch 시 TLB flush 안 함
```

리눅스 4.14+ 에서 사용. Meltdown 패치 (KPTI) 와 결합.

→ context switch 비용 ↓ 약 50%.

---

## 8. Cache 영향

```
스위치 직후 새 task 는 L1/L2 cold
첫 1000 instructions = cache miss 폭증
```

→ **CPU affinity** 로 같은 코어 유지하면 캐시 hot.

```bash
taskset -c 0 ./worker
```

---

## 9. NUMA

```
다른 NUMA node 로 마이그 → memory access cross-node → 3x 느림
```

→ NUMA-aware 스케줄링 (CFS 가 한다) / `numactl`.

```bash
numactl --membind=0 --cpunodebind=0 ./app
```

---

## 10. 컨테이너 / Kubernetes

CPU limit (cgroup quota) → throttle 발생 시 context switch + 휴식.

```bash
cat /sys/fs/cgroup/cpu.stat
# nr_throttled / throttled_usec
```

자세히 → [[../virtualization/cgroups]]

---

## 11. 줄이는 방법

### 11.1 더 적은 스레드
스레드 수 ≤ 코어 수 (CPU bound) 또는 측정 기반.

### 11.2 큰 timeslice
Latency 손해 vs throughput. CFS 의 latency target 조정.

### 11.3 CPU affinity
같은 코어에 고정 → 캐시 유지.

### 11.4 비동기 / lock-free
sleep / wait 줄임 → voluntary cs ↓.

### 11.5 NUMA pinning
같은 node 의 CPU + 메모리.

### 11.6 SO_REUSEPORT (network)
각 worker 가 자기 socket → cross-CPU bouncing X.

### 11.7 io_uring / epoll
syscall + cs 폭증 회피.

---

## 12. lmbench / sysbench

```bash
# lmbench
lat_ctx -P 1 2 4 8 16 32

# 결과 예시
# 2 procs, 0KB → 1.5 μs
# 16 procs, 64KB → 15 μs
```

cache size 가 큰 task = cs 비용 ↑ (cache thrashing).

---

## 13. Spectre / Meltdown 패치의 영향

- KPTI (Kernel Page Table Isolation) → context switch 시 page table 두 번 변경
- TLB flush 부담 ↑
- PCID 없으면 ~ 30% slower syscall / cs

→ 신규 CPU 의 PCID 지원이 mitigation.

---

## 14. 함정

### 14.1 너무 많은 스레드
cs 폭증. 풀 / 적정 수.

### 14.2 작은 timeslice 강제
context switch 폭발. CFS 의 latency 조정.

### 14.3 No CPU affinity
cross-CPU bouncing.

### 14.4 NUMA 무시
cross-node memory.

### 14.5 cgroup CPU limit 작음
throttle + cs 폭증.

### 14.6 Async + sync 혼합
의도와 다른 cs 패턴.

### 14.7 측정 단위 혼동
`pidstat` cs/s vs `perf stat` total count.

---

## 15. 학습 자료

- **OSTEP** Ch. 6
- **Linux Kernel Development** Ch. 4
- **Brendan Gregg — Systems Performance**
- **Tanenbaum** Ch. 2

---

## 16. 관련

- [[../process/pcb]] — task_struct
- [[cfs]]
- [[../virtualization/cgroups]]
- [[scheduling]] — Scheduling hub
