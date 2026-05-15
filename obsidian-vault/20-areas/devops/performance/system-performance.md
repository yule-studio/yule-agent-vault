---
title: "System performance — Brendan Gregg method"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:23:00+09:00
tags: [devops, performance, system, linux]
---

# System performance — Brendan Gregg method

**[[performance|↑ performance]]**

---

## 1. 60-Second Performance Analysis

```bash
uptime              # load average
dmesg | tail        # OOM kill, error
vmstat 1            # CPU / Mem / Swap
mpstat -P ALL 1     # per-core CPU
pidstat 1           # per-process CPU
iostat -xz 1        # disk IO
free -m             # memory
sar -n DEV 1        # network throughput
sar -n TCP,ETCP 1   # TCP error/retransmit
top                 # 또는 htop
```

→ Brendan Gregg 의 표준. 60초 안에 layer 파악.

---

## 2. CPU 분석

### load average

```
uptime
13:42:01 up 30 days, 21:23, 3 users, load average: 4.50, 3.20, 2.10
                                                    1m    5m   15m

cores=4:
  4.0 = 100% busy
  4.5 = 0.5 process 가 wait (over-saturated)
  
load avg = run-queue + uninterruptible wait
  → IO bound 도 load avg 올림
```

### top / vmstat

```
%us  user code
%sy  kernel (syscall)
%ni  nice
%id  idle
%wa  IO wait (★ disk 의심)
%hi  hard interrupt
%si  soft interrupt
%st  steal (VM hypervisor 가 가져감)
```

→ %sy 높음 = syscall 폭주. %st 높음 = 부족한 VM resource.

---

## 3. CPU per-process

```bash
pidstat 1
# Linux 5.4.0
# 13:42:00      UID       PID    %usr %system  %guest    %CPU   CPU  Command
# 13:42:01     1000      1234    50.0    10.0    0.0    60.0     2  java
# 13:42:01     1000      5678     5.0     2.0    0.0     7.0     3  redis-server

# 한 프로세스의 thread
pidstat -t -p <pid> 1

# perf top
sudo perf top -p <pid>
```

---

## 4. memory

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi        8.0Gi       1.0Gi     200Mi       6.0Gi       6.5Gi
# Swap:          2.0Gi          0B       2.0Gi

# 중요: available 이 실제 여유 (buff/cache 회수 가능)
# swap 사용 = 시작점
```

```bash
vmstat 1
#  r  b  swpd   free   buff   cache   si   so    bi    bo   in   cs  us sy id wa st
#  2  0     0  1234k  500k  6000k    0    0     0     0  1000 2000  60 10 30  0  0
#  ↑                                  ↑    ↑
#  run queue                         swap in/out (★ > 0 = 나쁨)
```

→ swap si/so > 0 = memory 부족 시작.

---

## 5. memory per-process

```bash
ps aux --sort=-%mem | head
# USER       PID  %CPU %MEM    VSZ    RSS  COMMAND
# user      1234   2.0  5.0  3000000  500000  java

# RSS  = 실제 사용 (resident)
# VSZ  = virtual size

# 자세히
pmap -x <pid>
cat /proc/<pid>/status | grep -E "Vm|Rss|Hugetlb"
```

---

## 6. disk IO

```bash
iostat -xz 1
# Device  rrqm/s wrqm/s   r/s   w/s   rkB/s   wkB/s  rareq-sz wareq-sz  await aqu-sz  %util
# nvme0n1   0.00   2.00 100.0  50.0  3000.0   500.0   30.0     10.0    5.20  0.80  80.0
#                                                                              ↑    ↑
#                                                                       latency  saturation %
```

→ `%util` 100% = saturated. `await` (ms) > 10ms 면 IO bound.

```bash
# per-process disk
iotop -o
# 또는 sudo cat /proc/<pid>/io
```

---

## 7. network

```bash
sar -n DEV 1
# 13:42:01  IFACE  rxpck/s  txpck/s   rxkB/s   txkB/s
# 13:42:02   eth0  10000     8000     1500     1200
#                  10K pkt/s            1.5MB/s

# 빠르게
ifstat
iftop
nethogs

# 통계
ss -s
nstat -az TcpRetransSegs
ss -i             # info (cwnd, rtt, retrans)
```

---

## 8. tcpdump (마지막 수단)

```bash
sudo tcpdump -i any -nn 'port 8080' -c 100
sudo tcpdump -i any -nn -A 'port 8080'   # ASCII payload
sudo tcpdump -w out.pcap -i any port 8080
# Wireshark 로 분석
```

---

## 9. open file / socket

```bash
lsof -p <pid> | wc -l           # 열린 파일 개수
lsof -i :8080                    # 8080 사용 process
lsof -i TCP -p <pid>             # process 의 TCP 연결

# /proc 직접
ls /proc/<pid>/fd/ | wc -l

# 한계
ulimit -n
cat /proc/<pid>/limits | grep "open files"
```

→ "Too many open files" → ulimit / fs.file-max ↑.

---

## 10. 흔한 진단 패턴

### "서버가 느리다"

```
1. uptime              load avg 높?
2. top                 CPU? mem?
3. iostat -x 1         disk?
4. dmesg | tail        OOM? error?
5. ss -s               TCP TIME_WAIT 많?
6. journalctl --since "1 hour ago"
```

### "deploy 후 OOM kill"

```
1. dmesg | grep -i "killed process"
2. /var/log/messages 또는 journalctl -k
3. cgroup memory limit?
4. JVM Xmx vs container limit
5. heap dump 분석
```

### "디스크 가득"

```
1. df -h                          mount?
2. du -sh /* | sort -h            top dir?
3. ncdu /var                      interactive
4. find / -size +1G -type f       big file
5. journalctl --disk-usage
6. docker system df               container?
```

### "thread 부족"

```
1. ps -eLf | wc -l                전체 thread
2. cat /proc/<pid>/status | grep Threads
3. ulimit -u                       max user processes
4. /proc/sys/kernel/threads-max
```

---

## 11. perf / eBPF (advanced)

```bash
# perf top
sudo perf top -p <pid>

# perf record + flame graph
sudo perf record -F 99 -p <pid> -g -- sleep 30
sudo perf script > out.perf

# eBPF (BCC tools)
sudo execsnoop      # exec syscall
sudo opensnoop      # open syscall  
sudo tcpconnect     # TCP connect
sudo biolatency     # block IO latency histogram
sudo offcputime -p <pid> 30   # 어디서 wait
sudo profile -p <pid> 30      # CPU sampling

# bpftrace
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_open* { @[comm] = count(); }'
```

→ Brendan Gregg 의 BCC / bpftrace tools 100+. system 디버깅의 best.

---

## 12. cgroup (container)

```bash
# v2 (현재 표준)
ls /sys/fs/cgroup/
systemd-cgls
systemd-cgtop

# 특정 process 의 cgroup
cat /proc/<pid>/cgroup

# container 의 cgroup
docker inspect <id> | grep -i cgroup
crictl inspect <id>

# k8s pod
crictl ps
crictl inspect <id> | jq '.info.runtimeSpec.linux.resources'
```

---

## 13. kernel parameter

```bash
# 현재 값
sysctl -a | grep <param>

# 자주 tuning:
# - net.core.somaxconn       listen backlog
# - net.ipv4.tcp_max_syn_backlog
# - net.ipv4.ip_local_port_range   ephemeral port
# - net.ipv4.tcp_tw_reuse          TIME_WAIT 재사용
# - net.netfilter.nf_conntrack_max
# - fs.file-max                    전체 fd
# - vm.swappiness                  swap 적극도
# - vm.dirty_ratio / dirty_background_ratio

# 영구
sudo nano /etc/sysctl.conf
sudo sysctl -p
```

---

## 14. NUMA

```bash
# multi-socket server 의 NUMA awareness
numactl --hardware
numactl --show
numastat -m

# JVM 의 NUMA
-XX:+UseNUMA

# DB 의 cross-NUMA 가 큰 성능 영향
```

→ 큰 메모리 server (192GB+) 에서 중요.

---

## 15. 함정

1. **`top` 만 보고 결론** — vmstat / iostat / ss 종합.
2. **`free` 의 used 만** — available 이 진짜.
3. **swap usage = 무조건 나쁨** 오해 — si/so 가 진짜.
4. **iostat 첫 줄** — boot 이후 평균. 둘째 줄부터.
5. **load avg = CPU 사용률** 오해 — IO wait 포함.
6. **strace prod** — process 느려짐.
7. **tcpdump 항상** — 디버깅 막바지.

---

## 16. 관련

- [[performance|↑ performance]]
- [[jvm-tuning]]
- [[profiling]]
- [[../linux/performance-troubleshooting|↗ linux performance]]
