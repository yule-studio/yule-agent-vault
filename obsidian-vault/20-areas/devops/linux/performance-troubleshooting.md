---
title: "Performance troubleshooting — CPU/Mem/Disk/IO"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:00:00+09:00
tags: [devops, linux, performance, troubleshooting]
---

# Performance troubleshooting — CPU/Mem/Disk/IO

**[[linux|↑ linux]]**

---

## 1. 1차 진단 (60초)

Brendan Gregg 의 60-Second Performance Analysis:

```bash
uptime              # load average
dmesg | tail        # 최근 kernel msg (OOM kill, error)
vmstat 1            # 1초마다 CPU / Mem
mpstat -P ALL 1     # core 별 CPU
pidstat 1           # process 별 CPU
iostat -xz 1        # disk IO
free -m             # memory
sar -n DEV 1        # network throughput
sar -n TCP,ETCP 1   # TCP stat
top                 # 또는 htop
```

→ 60초 안에 어디가 bottleneck 인지 감.

---

## 2. CPU 분석

```bash
# 사용률
top                          # %us %sy %wa %id
htop                         # 보기 좋음
mpstat -P ALL 1              # core 별

# 어떤 process?
ps aux --sort=-%cpu | head
pidstat 1                     # 실시간 per-process

# load average 의미
uptime
# load avg: 4.50, 3.20, 2.10   ← 1m, 5m, 15m
# 4-core 시 4.0 = 100% busy, 4.5 = 0.5 process 가 대기
```

| %us | user 코드 |
| %sy | kernel (syscall) |
| %wa | IO wait (★ 느리면 disk 의심) |
| %id | idle |
| %st | steal (VM hypervisor) |

→ **wa 가 높으면 disk IO 문제**, **sy 가 높으면 syscall 폭증**.

---

## 3. memory 분석

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi        8.0Gi       1.0Gi     200Mi       6.0Gi       6.5Gi
# Swap:          2.0Gi          0B       2.0Gi

# 중요: "available" 이 실제 여유 (buff/cache 회수 가능)

vmstat 1                     # si/so 가 0 이 아니면 swap (★ 나쁨)

# 어떤 process?
ps aux --sort=-%mem | head

# 자세히
pmap -x <pid>
cat /proc/<pid>/status | grep -i vm
cat /proc/meminfo
```

```bash
# OOM kill 확인
dmesg | grep -i "killed process"
journalctl -k | grep -i oom
```

→ swap 사용 ↑ 또는 OOM kill = memory 부족.

---

## 4. disk / IO 분석

```bash
iostat -xz 1                  # extended, busy device만
#  rrqm/s  wrqm/s  r/s  w/s   rkB/s   wkB/s  await  svctm  %util
#                                                          ↑ 100% 면 saturated

iotop                          # process 별 (root 필요)
iotop -o                       # 활동 중인 것만

# disk 사용
df -h                          # mount point 별
df -i                          # inode (작은 파일 많으면)
du -sh /var/log/*              # 디렉터리 별
ncdu /                         # interactive (★ 추천)

# slow query / IO
strace -c -p <pid>             # syscall 통계
```

`%util` 이 100% = disk 포화. `await` 가 크면 응답 지연.

---

## 5. network 분석

```bash
# throughput
sar -n DEV 1
iftop                         # interface 별 실시간
nethogs                       # process 별

# connection
ss -s                          # 통계 요약
ss -tn state established | wc -l
ss -tan state time-wait | wc -l   # TIME_WAIT 많으면 ephemeral port 부족

# 패킷
tcpdump -i any -nn
```

```bash
# TCP retransmit
ss -i                          # info (cwnd, rtt, retrans)
nstat -az TcpRetransSegs

# conntrack (NAT 환경)
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

---

## 6. 흔한 패턴별 진단

### "서버가 느리다"

```
1. uptime          → load avg ↑?
2. top             → CPU? memory?
3. iostat -x 1     → disk %util?
4. dmesg | tail    → OOM? error?
5. ss -s           → too many TIME_WAIT?
```

### "메모리 누수"

```
1. free -h         → swap 사용 시작?
2. ps --sort=-%mem → 점점 증가하는 process?
3. heap dump (Java: jmap)
4. /proc/<pid>/smaps → RSS 증가 추세
```

### "5xx 에러 spike"

```
1. log 확인 (Loki: ERROR)
2. 최근 deploy?
3. downstream 상태? (DB, Redis, 외부 API)
4. resource saturation? (CPU/Mem/Disk/Conn pool)
5. cardinality (cache key 폭주 등)
```

### "disk 가득"

```
1. df -h                      → 어느 mount?
2. du -sh /* | sort -h        → 어느 dir?
3. ncdu /var                  → interactive drill
4. find / -size +1G -type f   → 큰 파일
5. journalctl --disk-usage    → journal log?
6. docker system df           → container?
```

---

## 7. perf / eBPF (advanced)

```bash
# perf
perf top                       # 실시간 CPU
perf record -g -p <pid>        # profile
perf report                     # 결과

# eBPF (Brendan Gregg)
bpftrace -l                    # available probes
# 또는 BCC tools:
execsnoop                       # exec syscall
opensnoop                       # open syscall
tcpconnect                      # TCP connect
biolatency                      # block IO latency
```

→ kernel-level profiling. 고급 도구.

---

## 8. flame graph (★ Brendan Gregg)

```bash
# 1. perf 로 stack 수집
perf record -F 99 -p <pid> -g -- sleep 30
perf script > out.perf

# 2. FlameGraph (https://github.com/brendangregg/FlameGraph)
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > out.svg

# 3. 브라우저로 열기
```

→ CPU hot path 시각화. 디버깅의 최고 도구.

---

## 9. Java 특화

```bash
# heap
jmap -heap <pid>
jmap -histo:live <pid> | head -30
jmap -dump:format=b,file=heap.hprof <pid>

# GC
jstat -gc <pid> 1000           # 1초마다 GC
jstat -gcutil <pid> 1000

# thread
jstack <pid> > stack.txt
# stack.txt 분석: BLOCKED / WAITING 많으면 lock 문제

# 통합
jcmd <pid> Thread.print
jcmd <pid> GC.heap_info
jcmd <pid> VM.uptime
```

→ Java 서버는 G1GC / ZGC 등 GC 옵션도 영향.

---

## 10. cheat sheet

| 문제 | 첫 도구 |
| --- | --- |
| 느림 일반 | `top`, `htop` |
| CPU high | `pidstat 1`, `perf top` |
| memory | `free`, `ps --sort=-%mem` |
| swap 사용 | `vmstat 1` (si/so) |
| disk IO | `iostat -x 1`, `iotop` |
| disk 가득 | `df -h`, `ncdu` |
| network | `sar -n DEV`, `iftop`, `ss -s` |
| DNS | `dig`, `time curl` |
| TCP | `ss -tn`, `tcpdump` |
| Java | `jstack`, `jmap`, `jstat` |

---

## 11. 함정

1. **`top` 만 보고 cpu high** — load avg / wa / pidstat 종합.
2. **`free` 의 used 만 봄** — buff/cache 회수 가능, `available` 이 진짜.
3. **swap = 무조건 나쁨** 아님 — 약간 사용은 OK. `si/so` 증가가 진짜 나쁨.
4. **iostat 첫 줄은 boot 이후 평균** — 두 번째 줄부터 사용.
5. **load avg = CPU 사용률 아님** — disk wait 도 포함.
6. **strace prod 적용** — process 느려짐 (감수해야).

---

## 12. 관련

- [[linux|↑ linux]]
- [[process-management]]
- [[../monitoring/monitoring|↗ monitoring]]
