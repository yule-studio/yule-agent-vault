---
title: "strace / ltrace / perf — Tracing & Profiling"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:10:00+09:00
tags:
  - operating-system
  - linux
  - perf
  - tracing
---

# strace / ltrace / perf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 디버그 / 프로파일링 도구 |

**[[linux|↑ Linux hub]]**

---

strace 자세히 → [[../syscall/strace]]

여기는 **perf + 보조** 만.

---

## 1. perf — Linux Performance Tool

```bash
sudo apt install linux-tools-$(uname -r)
sudo apt install linux-tools-generic
```

`perf` = sampling-based profiler + hardware counter + tracing.

---

## 2. perf top — 실시간 hotspot

```bash
sudo perf top
sudo perf top -p $PID
sudo perf top -e cache-misses
sudo perf top -g                       # call graph
```

`top` 처럼 — CPU 시간이 어느 함수에 가는지.

---

## 3. perf stat — counter

```bash
sudo perf stat sleep 5
sudo perf stat -a sleep 5              # all CPUs
sudo perf stat -p $PID sleep 5
sudo perf stat -e cycles,instructions,cache-references,cache-misses ./app
sudo perf stat -e task-clock,context-switches,cpu-migrations,page-faults ./app
sudo perf stat -e branch-instructions,branch-misses ./app
sudo perf stat -e LLC-loads,LLC-load-misses ./app
sudo perf stat -e dTLB-loads,dTLB-load-misses ./app
```

### 3.1 핵심 지표

- **IPC** (Instructions Per Cycle) — 높을수록 좋음 (1.0+)
- **cache miss rate**
- **branch miss rate**
- **context switches**

```
1,234,567,890  cycles
1,500,000,000  instructions     #  1.21 insn per cycle
   10,000,000  cache-misses     #  2% miss
```

---

## 4. perf list

```bash
sudo perf list                          # 모든 event
sudo perf list cache
sudo perf list sched:
```

수백 개 event — hardware (PMU), tracepoint, kprobe, uprobe.

---

## 5. perf record + report — flame graph

```bash
sudo perf record -F 99 -a -g -- sleep 30
# F 99 = 99 Hz sampling
# -a = all CPU
# -g = call graph

sudo perf report
sudo perf report --no-children          # CPU 자신만
```

### 5.1 Flame Graph

```bash
git clone https://github.com/brendangregg/FlameGraph

sudo perf record -F 99 -a -g -- sleep 60
sudo perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg
```

→ x = 시간 비율, y = 스택 깊이. CPU bottleneck 시각화.

Brendan Gregg 의 대표 작품.

---

## 6. perf sched — 스케줄러

```bash
sudo perf sched record -- sleep 30
sudo perf sched latency
sudo perf sched timehist
```

run queue latency / context switch 분석.

---

## 7. perf trace — strace 대체

```bash
sudo perf trace ./app
sudo perf trace -p $PID
sudo perf trace -s ./app                # summary
```

strace 보다 overhead ↓ (eBPF 기반).

---

## 8. perf c2c — cache contention

```bash
sudo perf c2c record sleep 30
sudo perf c2c report
```

False sharing / cross-socket cache miss 찾기.

자세히 → [[../synchronization/atomic#12-false-sharing]]

---

## 9. perf mem — memory access

```bash
sudo perf mem record ./app
sudo perf mem report
```

NUMA / cache 분석.

---

## 10. perf probe — 동적 tracepoint

```bash
sudo perf probe -x ./app 'do_work argc:s32 path:string'
sudo perf record -e probe_app:do_work ./app
```

debug symbol 활용. 응용 함수에 동적 tracepoint.

---

## 11. ltrace — Library call

```bash
ltrace ./app
ltrace -c ./app                          # summary
ltrace -p $PID
```

`malloc / printf / strcpy` 등 — userspace library 의 호출.

strace = syscall (커널 진입), ltrace = library function.

---

## 12. /proc/$PID/stack — kernel stack

```bash
sudo cat /proc/$PID/stack
# [<ffffffff...>] futex_wait
# [<ffffffff...>] do_syscall_64
```

D state 의 process — 어디서 sleep 중인지.

---

## 13. eBPF / bpftrace

자세히 → [[ebpf]]

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'

# bcc-tools
sudo opensnoop
sudo execsnoop
sudo biolatency
```

→ perf 의 후계. 더 강력 + 안전.

---

## 14. valgrind

```bash
valgrind --leak-check=full ./app
valgrind --tool=memcheck ./app
valgrind --tool=callgrind ./app          # profiling
valgrind --tool=cachegrind ./app
```

매우 느림 (10x+). dev / CI 만.

---

## 15. heaptrack / massif

```bash
heaptrack ./app
heaptrack_gui heaptrack.out
```

heap 사용량 시각화.

---

## 16. gdb

```bash
gdb -p $PID
gdb ./app core
```

```
(gdb) bt full
(gdb) thread apply all bt
(gdb) info threads
(gdb) frame 3
(gdb) print var
(gdb) watch var
```

자세히 → [[../tools/gdb]] (작성 예정)

---

## 17. SystemTap / DTrace / ftrace

| | 설명 |
| --- | --- |
| **ftrace** | Linux 내장, `/sys/kernel/debug/tracing/` |
| **SystemTap** | Linux 동적 instrumentation (옛) |
| **DTrace** | Solaris/macOS/FreeBSD — Linux 도 port |

대부분 eBPF 가 대체.

---

## 18. 실전 시나리오

### 18.1 CPU 100% — 어디서?

```bash
sudo perf top -p $PID
# 또는 flame graph
sudo perf record -F 99 -p $PID -g -- sleep 30
sudo perf script | ... | flamegraph.pl > flame.svg
```

### 18.2 응용이 느림

```bash
sudo perf stat -p $PID sleep 30
# IPC 낮음 → cache miss?
sudo perf stat -e cache-references,cache-misses -p $PID sleep 30
```

### 18.3 hang process

```bash
sudo cat /proc/$PID/stack
sudo cat /proc/$PID/wchan
gdb -p $PID
(gdb) thread apply all bt
```

### 18.4 syscall 폭증

```bash
strace -c ./app                          # summary
sudo perf trace -s ./app
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'
```

### 18.5 디스크 I/O 느림

```bash
sudo iostat -x 1
sudo biolatency-bpfcc
sudo iotop
```

### 18.6 메모리 leak

```bash
valgrind --leak-check=full ./app
heaptrack ./app
sudo bpftrace tools/memleak.bt
```

---

## 19. perf 의 overhead

- sampling = 적음 (1-5%)
- tracepoint = 보통
- kprobe / uprobe = 더 많음
- bpftrace = 효율적

생산 환경에 사용 가능 (단, recording 시간 짧게).

---

## 20. 함정

### 20.1 perf 권한
`kernel.perf_event_paranoid` 가 너무 높으면 일반 user 권한 X.
`-1` = root 만, `0` = kernel access OK, `1` = process 만, `2` = 더 제한 (기본).

### 20.2 debug symbol 없음
flame graph 가 `0x12345` 만. `-g` build + `objcopy --only-keep-debug` / debuginfo 패키지.

### 20.3 strace 의 overhead
응용 매우 느려짐. perf trace / bpftrace.

### 20.4 sampling rate
99 Hz vs 1000 Hz — overhead vs 정밀.

### 20.5 단기 sample
30s 으로 hot path 못 잡을 수도. 의심되는 시점에 record.

### 20.6 JIT 코드의 symbol
JVM / V8 의 JIT → perf 가 symbol 모름. perf-map-agent / `--mappings`.

### 20.7 컨테이너 안 trace
host 의 perf 가 container 안 process trace → 가능, 단 user_namespace mapping 주의.

---

## 21. 학습 자료

- **Systems Performance** — Brendan Gregg (필독)
- **brendangregg.com** — flame graph + USE
- `perf(1)` man pages
- **Linux Kernel — perf Documentation/admin-guide/perf-tools.rst**
- **BPF Performance Tools** — Brendan Gregg

---

## 22. 관련

- [[ebpf]]
- [[../syscall/strace]]
- [[monitoring]]
- [[linux]] — Linux hub
