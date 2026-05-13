---
title: "perf — Linux Performance Counter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:15:00+09:00
tags:
  - operating-system
  - tools
  - perf
---

# perf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | perf 사용 가이드 |

**[[tools|↑ Tools hub]]**

---

perf 의 자세히 → [[../linux/strace-perf]]

여기는 **빠른 cheatsheet** 만.

---

## 1. 설치

```bash
sudo apt install linux-tools-$(uname -r) linux-tools-generic
```

권한:
```bash
# kernel.perf_event_paranoid
# -1 root only events 도 OK
#  0 cpu events OK
#  1 user code OK (기본)
#  2 더 제한
sudo sysctl kernel.perf_event_paranoid=1
```

---

## 2. 가장 자주 쓰는 명령

```bash
# CPU 사용 hotspot
sudo perf top
sudo perf top -p $PID

# event 통계
sudo perf stat -a sleep 5
sudo perf stat -p $PID sleep 10
sudo perf stat -e cycles,instructions,cache-references,cache-misses ./app

# 프로파일 record
sudo perf record -F 99 -a -g -- sleep 30
sudo perf record -F 99 -p $PID -g -- sleep 30
sudo perf report
sudo perf report --sort=symbol
sudo perf annotate

# 사용 가능 event 목록
sudo perf list | less
```

---

## 3. Flame Graph

```bash
git clone https://github.com/brendangregg/FlameGraph

sudo perf record -F 99 -a -g -- sleep 30
sudo perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > out.svg
```

→ x = 시간 비율, y = 스택 깊이. hotspot 찾기.

대안: `pprof` (Go), `py-spy` (Python), `async-profiler` (Java).

---

## 4. perf trace — strace 대체

```bash
sudo perf trace ./app                       # 비슷한 strace
sudo perf trace -p $PID
sudo perf trace -s ./app                    # summary
sudo perf trace --pf maj                     # major fault 만
```

overhead 가 strace 보다 ↓.

---

## 5. perf sched

```bash
sudo perf sched record -- sleep 30
sudo perf sched latency
sudo perf sched timehist
```

run queue / context switch 분석.

---

## 6. perf c2c (cache-to-cache)

```bash
sudo perf c2c record sleep 30
sudo perf c2c report
```

False sharing / cache line bouncing.

자세히 → [[../synchronization/atomic#12-false-sharing]]

---

## 7. perf mem

```bash
sudo perf mem record ./app
sudo perf mem report
```

NUMA / cache 분석.

---

## 8. perf probe

```bash
# kernel function tracepoint
sudo perf probe --add 'do_sys_open filename:string'
sudo perf record -e probe:do_sys_open -a sleep 10
sudo perf script

# user function
sudo perf probe -x ./app 'work argc:s32'
```

---

## 9. perf list

```bash
sudo perf list                              # 모든 event
sudo perf list cache
sudo perf list sched
sudo perf list 'syscalls:*'
```

수백 — HW (PMU) + tracepoint + kprobe + uprobe.

---

## 10. 자주 분석하는 event

| Event | 의미 |
| --- | --- |
| `cycles` | CPU cycle |
| `instructions` | 명령 수 → IPC |
| `cache-references / misses` | LLC |
| `L1-dcache-loads / load-misses` | L1 |
| `dTLB-loads / load-misses` | TLB |
| `branches / branch-misses` | branch predictor |
| `context-switches` | cs / s |
| `cpu-migrations` | core 이동 |
| `page-faults / major-faults` | 메모리 |
| `task-clock` | CPU time |

```bash
sudo perf stat -e cycles,instructions ./app
# → IPC (Instructions Per Cycle) 계산
```

IPC > 1.0 → 좋음.

---

## 11. 측정 예

### 11.1 응용 IPC

```
sudo perf stat -e cycles,instructions ./app
   1,234,567,890 cycles
   1,500,000,000 instructions     # 1.21 insn per cycle
```

### 11.2 cache miss rate

```
sudo perf stat -e cache-references,cache-misses ./app
   100,000,000 cache-references
     5,000,000 cache-misses       # 5%
```

### 11.3 TLB miss

```
sudo perf stat -e dTLB-loads,dTLB-load-misses ./app
```

TLB miss 높으면 → Huge Page 검토.
자세히 → [[../memory/huge-pages]]

### 11.4 branch miss

```
sudo perf stat -e branches,branch-misses ./app
```

높으면 — branch predictor 의 한계. branchless code 검토.

---

## 12. perf 의 overhead

- sampling = 적음 (1-5%)
- tracepoint = 보통
- bpf 변환 = 보통
- kprobe / uprobe = 더 큼

대부분 운영에 안전. 단, recording 시간 짧게.

---

## 13. JIT 코드의 symbol

JVM / V8 / .NET → JIT — perf 가 symbol 모름.

```bash
# JVM
java -XX:+PreserveFramePointer -agentpath:libperf-jvmti.so ./app
# /tmp/perf-<pid>.map 생성 → perf 가 자동 찾음

# Node
node --perf-basic-prof ...
```

`perf-map-agent` (Brendan Gregg).

---

## 14. JIT / FP — Frame Pointer

```bash
# -fno-omit-frame-pointer 로 컴파일 (Linux 11+ default)
gcc -fno-omit-frame-pointer -g main.c
```

stack walk 가 정확해짐 → flame graph 완성.

dwarf unwinding (`--call-graph dwarf`) 대안.

---

## 15. 함정

### 15.1 권한
`kernel.perf_event_paranoid` 의 값.

### 15.2 debug symbol 누락
flame graph 가 `0x12345` 만 표시. `-g` build + debuginfo.

### 15.3 sampling rate
99 Hz vs 1000 Hz — overhead vs 정밀.

### 15.4 짧은 sample
30s 으로 hot path 못 잡을 수도.

### 15.5 frame pointer 누락
stack 깊이 부정확. `-fno-omit-frame-pointer` 또는 dwarf.

### 15.6 container 안 perf
host perf 가 container process trace → 가능. user_namespace mapping.

### 15.7 PMU 한계
HW counter 수 (보통 4-8). 많이 동시 측정 시 multiplex.

### 15.8 eBPF 대안
새 도구는 eBPF — bcc / bpftrace.

---

## 16. 자세히

[[../linux/strace-perf]] — 사용 시나리오.

---

## 17. 학습 자료

- **brendangregg.com/perf.html** — 가장 좋음
- **Systems Performance** — Brendan Gregg
- **perf wiki** — perf.wiki.kernel.org

---

## 18. 관련

- [[../linux/strace-perf]]
- [[../linux/ebpf]]
- [[gdb]]
- [[tools]] — Tools hub
