---
title: "NUMA — Non-Uniform Memory Access"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:15:00+09:00
tags:
  - operating-system
  - memory
  - numa
---

# NUMA

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | NUMA + binding |

**[[memory|↑ Memory hub]]</br>**

---

## 1. 한 줄

여러 CPU socket 시스템에서, 각 CPU 가 가까운 메모리 (local) 와 먼 메모리 (remote) 를 가짐. 같은 RAM 이라도 access 비용 다름.

```
local DRAM:  ~ 100 ns
remote (다른 socket): ~ 250-400 ns
```

→ 무시하면 1.5-3x 느림.

---

## 2. 토폴로지 확인

```bash
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0-15
# node 0 size: 32 GB
# node 0 free: 5 GB
# node 1 cpus: 16-31
# node 1 size: 32 GB
# distance 0-0: 10, 0-1: 21

lscpu                       # NUMA node 정보
hwloc-ls / lstopo            # 시각화
```

---

## 3. /proc/zoneinfo + /sys/devices/system/node/

```bash
cat /sys/devices/system/node/node0/cpulist
cat /sys/devices/system/node/node0/meminfo
```

---

## 4. NUMA 정책

```bash
numactl --membind=0 --cpunodebind=0 ./app
numactl --interleave=all ./app
numactl --preferred=1 ./app
numactl --localalloc ./app
```

| 정책 | 의미 |
| --- | --- |
| `default` | local 우선 |
| `bind` | 지정 node 만 — 부족 시 OOM |
| `preferred` | 우선, 부족 시 다른 |
| `interleave` | round-robin |
| `localalloc` | thread 가 실행되는 node 의 메모리 |

---

## 5. 응용

```c
#include <numa.h>

if (numa_available() < 0) ...;
int node = numa_node_of_cpu(sched_getcpu());

void *p = numa_alloc_onnode(sz, node);
numa_free(p, sz);

// 명시 마이그
numa_move_pages(...);
```

`-lnuma`.

---

## 6. 자동 NUMA Balancing

Linux 가 정기적으로 cross-node access 측정 → 페이지를 access 자주 하는 node 로 마이그.

```bash
cat /proc/sys/kernel/numa_balancing
# 0 / 1
```

대부분의 워크로드에 도움. 하지만 마이그 자체 비용도 있음 → DB 등 latency 민감은 끄기도.

---

## 7. /proc/$PID/numa_maps

```
55b1... default file=/bin/cat ...
7f4a... interleave:0-1 anon=234 ...
```

각 VMA 의 NUMA 정책 + page 분포.

---

## 8. JVM / DB 의 NUMA

### 8.1 JVM

```bash
-XX:+UseNUMA
```

heap 을 node 별 chunk 로 분할 + thread affinity.

### 8.2 PostgreSQL

권장: numactl --interleave=all + 큰 shared_buffers 가 한 node 에 몰리지 않게.

### 8.3 MySQL InnoDB

`innodb_numa_interleave = ON` (5.6.27+).

### 8.4 Redis
보통 단일 node 에 pin.

---

## 9. 측정

```bash
numastat
# node0     node1
# numa_hit  ...   ...
# numa_miss ...   ...      ← local 의도 but remote 사용
# numa_foreign      ...    ← remote 의 페이지가 local 에서 사용
# interleave_hit
# local_node
# other_node

numastat -p $PID
numastat -m            # 전체

# 도구
perf c2c                 # cache-to-cache (false sharing 분석)
perf stat -e node-loads,node-load-misses ./app
```

---

## 10. NUMA-aware 디자인

- **thread pool 을 socket 별 분리**
  - worker N per socket
- **메모리 alloc 을 같은 node 에서**
- **lock / atomic 변수 cross-node 회피**
- **SO_REUSEPORT + bind to node** (network server)

---

## 11. AWS / 클라우드

대형 인스턴스 (예: c6i.32xlarge) 는 NUMA 2-4 node. m6a.* 는 더 큰.
NUMA 무시 = 30%+ 성능 손실 가능.

---

## 12. 함정

### 12.1 NUMA 무시
cross-node access 폭증.

### 12.2 numactl --interleave 의존
average OK 지만 worst case 일정 X.

### 12.3 cgroup CPU + NUMA mismatch
CPU 는 node0 인데 memory 는 node1 → cross.

### 12.4 자동 balancing 의 overhead
페이지 마이그 비용. 일부 워크로드 비활성.

### 12.5 큰 JVM heap 의 NUMA-unaware
`-XX:+UseNUMA` 누락.

### 12.6 huge page + NUMA
node 별 reserve 누락 → 한쪽 만 사용.

### 12.7 false sharing + NUMA
cache line 의 cross-node bouncing → 극도로 느림.

---

## 13. 학습 자료

- **What Every Programmer Should Know About Memory** — Drepper
- `Documentation/vm/numa.rst`
- **Linux NUMA Best Practices** — Red Hat / SUSE 문서
- **brendangregg.com — NUMA**

---

## 14. 관련

- [[../scheduling/cfs]] — NUMA balancing
- [[allocators]]
- [[memory]] — Memory hub
