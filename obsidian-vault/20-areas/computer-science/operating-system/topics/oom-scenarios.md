---
title: "OOM 시나리오 — 메모리 부족의 모든 케이스"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:30:00+09:00
tags:
  - operating-system
  - topic
  - oom
---

# OOM 시나리오

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | OOM 종류 + 디버그 |

**[[topics|↑ Topics hub]]**

---

OOM 의 기본 메커니즘 → [[../memory/swap-oom]]

여기는 **실무 시나리오 / 디버그** 중심.

---

## 1. 종류

| OOM 종류 | 시나리오 |
| --- | --- |
| **System OOM** | 호스트 RAM + swap 부족 |
| **cgroup OOM** | container / service 의 memory.max 초과 |
| **응용 자체 OOM** | JVM heap exceeded / Python MemoryError |
| **fork OOM** | overcommit + 큰 fork 시도 (Redis BGSAVE) |
| **VMA limit** | vm.max_map_count 초과 |
| **FD limit** | nofile (메모리 X 지만 비슷한 limit error) |
| **task limit** | TasksMax / pids.max |

---

## 2. System OOM Killer

```
RAM + swap 모두 부족 → 새 page fault 시 alloc 실패
→ OOM Killer 가 점수 (oom_score) 높은 process SIGKILL
```

```bash
dmesg | grep -i 'killed process'
journalctl -k --grep -i oom

# 결과 예
Out of memory: Killed process 1234 (mysqld) total-vm:... anon-rss:...
```

자세히 → [[../memory/swap-oom]]

---

## 3. cgroup OOM (Container / K8s)

```bash
# 호스트 dmesg
dmesg | grep -i 'memory cgroup out of memory'

# cgroup 자체
cat /sys/fs/cgroup/.../memory.events
# oom 5
# oom_kill 5
# max 100
```

K8s:
```bash
kubectl describe pod mypod | grep -A3 OOMKilled
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

→ container 의 RSS 가 memory limit 초과.

해결:
- limit 늘림
- 응용 메모리 leak 잡기
- JVM `-Xmx` 와 limit 차이 (limit ≥ heap + GC overhead + native + threads stack)

---

## 4. JVM OOM

```
java.lang.OutOfMemoryError: Java heap space
java.lang.OutOfMemoryError: Metaspace
java.lang.OutOfMemoryError: GC overhead limit exceeded
java.lang.OutOfMemoryError: unable to create new native thread
java.lang.OutOfMemoryError: Direct buffer memory
```

### 4.1 Heap

```bash
java -Xmx4g ./app
```

→ heap 안에서 alloc 실패. heap dump 분석.

```bash
jcmd $PID GC.heap_dump /tmp/heap.hprof
# 또는 자동
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/dump/

# 분석 — Eclipse MAT, jhat, VisualVM
```

### 4.2 Metaspace
class 로딩 폭증 (옛 Permgen 후계).

```bash
-XX:MaxMetaspaceSize=512m
```

### 4.3 Native
스레드 수 폭증 (ulimit) 또는 native buffer.

### 4.4 Container OOM (heap < limit 이지만 OOM)
```
JVM heap = 2G
+ GC overhead ~ 200 MB
+ Metaspace ~ 256 MB
+ thread stack × N
+ off-heap (NIO direct)
+ JIT code cache
= 3 GB+ 가능
```

→ container limit 을 heap + 50% 권장.

---

## 5. Python MemoryError

```python
MemoryError
```

- 큰 list / dict
- 무한 generator → list()
- 큰 numpy / pandas

```python
import resource
resource.setrlimit(resource.RLIMIT_AS, (4 * 1024**3, 4 * 1024**3))    # 4GB

import tracemalloc
tracemalloc.start()
# ...
top = tracemalloc.take_snapshot().statistics('lineno')[:10]
```

---

## 6. fork + COW + Redis 의 함정

```
Redis 의 BGSAVE / BGREWRITEAOF
→ fork()
→ 부모가 계속 write → CoW 페이지 복사 → 메모리 폭증
```

해결:
```ini
# /etc/sysctl.conf
vm.overcommit_memory=1
```

자세히 → [[../../database/redis/persistence#7-fork-의-위험]]

---

## 7. vm.max_map_count

Elasticsearch / 큰 mmap 응용:

```
java.io.IOException: Map failed
```

원인 — `vm.max_map_count` 기본 65530 부족.

해결:
```bash
sysctl vm.max_map_count=262144
```

`/etc/sysctl.d/99-elasticsearch.conf`:
```
vm.max_map_count=262144
```

자세히 → [[../memory/mmap]]

---

## 8. ulimit nofile — "Too many open files"

메모리 X 지만 비슷한 자원 한계:

```
accept: Too many open files
EMFILE — process limit
ENFILE — system limit
```

```bash
ulimit -n
ulimit -n 65535

# 영구
# /etc/security/limits.conf
*  soft  nofile  65535
*  hard  nofile  65535
```

systemd:
```ini
LimitNOFILE=65535
```

---

## 9. PIDs / Task limit

```
fork: Resource temporarily unavailable
EAGAIN — nproc limit
```

```bash
ulimit -u 65535
sysctl kernel.pid_max=4194304
```

cgroup pids.max → K8s 의 pod 안 thread 수 한계.

---

## 10. Zombie 누적

```
PID 공간 32768 → 모두 zombie 점령 → fork 실패
```

자세히 → [[../process/orphan-zombie]]

container PID 1 이 reap 안 함 — `--init` / tini.

---

## 11. 진단 흐름

```bash
# 1. 시스템 전체
free -h
cat /proc/meminfo
vmstat 1
dmesg | grep -i oom
journalctl -k --since today

# 2. 어떤 process
ps aux --sort=-rss | head
# 또는 smem (PSS / USS)
smem -tk

# 3. 그 process 상세
cat /proc/$PID/status | grep Vm
cat /proc/$PID/smaps
pmap -x $PID

# 4. cgroup
systemd-cgtop
cat /sys/fs/cgroup/.../memory.current
cat /sys/fs/cgroup/.../memory.events

# 5. swap
swapon --show
sar -W 1
```

---

## 12. 디버그 시나리오

### 12.1 매일 새벽 OOM

```
원인 — log rotation / cron / GC pause / backup
journalctl --since "yesterday 03:00" --until "yesterday 04:00"
```

### 12.2 점진적 RSS 증가 (leak)

```
RSS 천천히 ↑ → 결국 OOM
→ heap dump / valgrind / pprof / py-spy
```

자세히 → [[../linux/strace-perf]]

### 12.3 단기 spike OOM

```
일시 RSS 폭증 → 즉시 OOM
→ batch 처리, fork, 큰 query, file load
```

해결: streaming / chunking / spill-to-disk.

### 12.4 cgroup OOM 만

```
container limit 잘못 — JVM heap > limit
```

---

## 13. 보호 / 회피

### 13.1 swap 추가 (작게 — 안전망)
```bash
fallocate -l 2G /swap
mkswap /swap
swapon /swap
```

DB 는 swap 작게 (1-2 GB) + swappiness=1.

### 13.2 oom_score_adj
중요 process 보호:
```bash
echo -500 > /proc/$PID/oom_score_adj
```

systemd:
```ini
OOMScoreAdjust=-500
```

### 13.3 systemd-oomd / earlyoom
PSI 기반 조기 OOM. user space.

### 13.4 cgroup memory.high
hard limit (`memory.max`) 전에 reclaim 적극.

```ini
MemoryHigh=3G
MemoryMax=4G
```

자세히 → [[../virtualization/cgroups]]

### 13.5 정기 restart
응용 leak 이 잡히지 않을 때 — 매일 새벽 restart.
```
RestartSec=3600
RuntimeMaxSec=86400        # 24시간 후 재시작
```

---

## 14. K8s 의 OOM

| 종류 | 의미 |
| --- | --- |
| Pod OOMKilled | container 가 limit 초과 |
| Node OOM | host 의 OS OOM Killer |
| eviction | kubelet 이 자원 부족 시 pod 추출 |

```bash
kubectl describe pod x | grep -A3 'OOMKilled\|Evicted'
kubectl top pod
kubectl top node
```

K8s 의 QoS class:
- Guaranteed (request = limit)
- Burstable
- BestEffort (request 없음 — 가장 먼저 죽음)

---

## 15. 함정

### 15.1 RSS 만 보고
실제 사용 = PSS (공유 분배). `smem`.

### 15.2 free 의 "used"
page cache 포함. `available` 봐야.

### 15.3 swap 켜고 안심
thrashing 위험. 작은 swap + monitoring.

### 15.4 container limit 만
JVM heap + overhead. limit ≥ heap × 1.5.

### 15.5 zombie / fork OOM 누락
SIGCHLD reap + max_map_count + nproc.

### 15.6 OOM Killer 가 wrong process kill
oom_score_adj 명시.

### 15.7 silent OOM
swap thrashing → 사실상 hang. PSI 모니터링.

### 15.8 container restart loop
응용이 시작하자마자 OOM. limit 늘리거나 응용 분석.

---

## 16. 학습 자료

- **OOM Killer 코드** — `mm/oom_kill.c`
- **Brendan Gregg — Memory** 시리즈
- **K8s OOM debugging** — official docs
- **JVM Memory Tuning**

---

## 17. 관련

- [[../memory/swap-oom]]
- [[../memory/virtual-memory]]
- [[../virtualization/cgroups]]
- [[topics]] — Topics hub
