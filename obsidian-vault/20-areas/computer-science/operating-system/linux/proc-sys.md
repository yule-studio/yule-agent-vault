---
title: "/proc / /sys — Virtual Filesystem"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:05:00+09:00
tags:
  - operating-system
  - linux
  - proc
  - sysfs
---

# /proc / /sys

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | procfs + sysfs |

**[[linux|↑ Linux hub]]**

---

## 1. /proc — Process FS

```
가상 — 메모리에서 즉시 생성
1 process = 1 디렉토리 (/proc/$PID/)
+ 시스템 전반 (/proc/meminfo, /proc/cpuinfo, ...)
```

---

## 2. /proc/$PID/

```bash
ls /proc/$$/
```

| 파일 | 의미 |
| --- | --- |
| `cmdline` | command (null 구분) |
| `comm` | command name (short) |
| `status` | 가독성 정보 (state, VmRSS, ...) |
| `stat` | raw 수치 (top 이 파싱) |
| `statm` | memory |
| `maps` | 메모리 맵 |
| `smaps` | 자세한 메모리 |
| `pagemap` | 가상 → 물리 매핑 (root) |
| `fd/` | 열린 FD |
| `fdinfo/` | FD 정보 (epoll 등) |
| `environ` | env (null 구분) |
| `cwd` | symlink → current dir |
| `exe` | symlink → binary |
| `root` | symlink → root (chroot) |
| `task/` | 스레드 |
| `ns/` | namespace |
| `limits` | rlimit |
| `sched` | scheduler 상태 |
| `stack` | kernel stack (D state 디버그) |
| `cgroup` | cgroup 소속 |
| `oom_score / oom_score_adj` | OOM |
| `io` | I/O 통계 |
| `mounts` | mount table |
| `mountinfo` | 더 자세 |
| `net/` | network namespace 의 통계 |

자주 쓰는 패턴:

```bash
cat /proc/$$/cmdline | tr '\0' ' '
cat /proc/$$/status | grep VmRSS
cat /proc/$PID/environ | tr '\0' '\n'
ls -l /proc/$PID/fd | head
sudo cat /proc/$PID/stack            # D state 의 kernel
cat /proc/$PID/limits
cat /proc/$PID/cgroup
```

자세히 → [[../process/pcb#7-procpid-으로-보기]]

---

## 3. /proc 시스템 전반

| 파일 | 의미 |
| --- | --- |
| `cpuinfo` | CPU 정보 |
| `meminfo` | 메모리 |
| `loadavg` | load |
| `uptime` | 부팅 후 시간 + idle |
| `version` | kernel |
| `cmdline` | kernel boot params |
| `mounts` | 마운트 |
| `swaps` | swap |
| `partitions` | 파티션 |
| `interrupts` | IRQ |
| `softirqs` | softirq |
| `stat` | system 누적 (idle, ctxt, ...) |
| `vmstat` | vm 통계 |
| `slabinfo` | kernel slab |
| `modules` | loaded modules |
| `kallsyms` | kernel symbols (root) |
| `buddyinfo` | buddy allocator |
| `zoneinfo` | NUMA zones |
| `net/dev` | NIC bytes |
| `net/tcp / net/udp / net/unix` | sockets |
| `net/route / net/arp` | net |
| `sys/...` | sysctl |
| `pressure/cpu / memory / io` | PSI |

```bash
cat /proc/cpuinfo | grep 'model name' | head -1
cat /proc/meminfo | head
cat /proc/loadavg
cat /proc/uptime
cat /proc/sys/kernel/hostname
```

---

## 4. /proc/sys (sysctl)

```bash
sysctl -a                              # 모두
sysctl kernel.hostname
sysctl vm.swappiness
sysctl net.ipv4.ip_forward=1
sysctl -p                              # /etc/sysctl.conf 적용
sysctl --system                        # /etc/sysctl.d/* 모두
```

영구:
```bash
# /etc/sysctl.d/99-mine.conf
vm.swappiness=10
net.core.somaxconn=4096
```

자주 변경:
- `vm.swappiness`
- `vm.overcommit_memory`
- `vm.max_map_count`
- `kernel.pid_max`
- `kernel.panic`
- `net.core.somaxconn`
- `net.ipv4.tcp_*`
- `fs.file-max`
- `fs.inotify.max_user_watches`

자세히 → [[tuning]]

---

## 5. /sys — Sysfs

```
디바이스 / 커널 객체 트리
/sys/class/   — 분류별 (net, block, gpu, ...)
/sys/devices/  — 물리 토폴로지
/sys/bus/      — bus (pci, usb, ...)
/sys/block/    — 블록 디바이스
/sys/fs/       — fs 통계
/sys/kernel/   — kernel
/sys/module/   — 로드된 module
/sys/firmware/
/sys/power/    — 전원
```

```bash
# 네트워크
cat /sys/class/net/eth0/operstate
cat /sys/class/net/eth0/mtu
cat /sys/class/net/eth0/statistics/rx_bytes

# 디스크
cat /sys/block/sda/size
cat /sys/block/sda/queue/scheduler
echo none > /sys/block/sda/queue/scheduler

# CPU
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# THP
cat /sys/kernel/mm/transparent_hugepage/enabled

# cgroup v2
ls /sys/fs/cgroup/

# CPU offline
echo 0 > /sys/devices/system/cpu/cpu5/online
```

---

## 6. /proc/<pid>/ns — Namespace 확인

```bash
ls -l /proc/$$/ns/
# cgroup -> 'cgroup:[40259...]'
# ipc    -> 'ipc:[40259...]'
# mnt    -> 'mnt:[40259...]'
# net    -> 'net:[40259...]'
# pid    -> 'pid:[40259...]'
# user   -> 'user:[40258...]'
# uts    -> 'uts:[40259...]'

# 같은 inode = 같은 namespace
lsns -p $$
```

자세히 → [[../virtualization/namespace]]

---

## 7. /proc/net

```bash
cat /proc/net/tcp                       # IPv4 TCP socket
cat /proc/net/tcp6
cat /proc/net/udp
cat /proc/net/unix                       # Unix socket
cat /proc/net/dev                        # NIC 통계
cat /proc/net/route                       # routing table
cat /proc/net/arp                         # ARP cache
cat /proc/net/netstat
cat /proc/net/sockstat                    # socket count
```

`ss` / `ip` 가 이걸 파싱.

---

## 8. /proc/$PID/fd vs /proc/$PID/fdinfo

```bash
ls -l /proc/$PID/fd/
# 0 -> /dev/pts/3
# 1 -> /dev/pts/3
# 2 -> /dev/pts/3
# 3 -> /etc/hosts
# 4 -> 'socket:[12345]'
# 5 -> 'pipe:[67890]'

cat /proc/$PID/fdinfo/3
# pos: 1234
# flags: 02
# mnt_id: 21
```

→ 응용이 어떤 파일 / socket / pipe 잡고 있나.

`lsof -p $PID` 가 친화적.

---

## 9. /dev/kmsg + dmesg

```bash
dmesg
dmesg -w
dmesg -T

cat /dev/kmsg                            # raw

# 권한 (4.10+)
sysctl kernel.dmesg_restrict
```

kernel 의 ring buffer — printk 출력.

---

## 10. /proc/$PID/io

```bash
cat /proc/$PID/io
# rchar: 1234567        sys 가 user 에 read 한 byte
# wchar: 654321
# syscr: 100             read syscall 횟수
# syscw: 50
# read_bytes: 12345     실제 디스크 read
# write_bytes: 6789
# cancelled_write_bytes: 0
```

`pidstat -d` 가 친화적.

---

## 11. /proc/buddyinfo / /proc/slabinfo

```bash
cat /proc/buddyinfo
# Node 0, Zone Normal     500   200   100   50   ...
# (order 0, 1, 2 ... 의 free block 수)

sudo cat /proc/slabinfo | head
# kmalloc-32         12345  20000   32   ...
# task_struct         200    500  1024   ...

slabtop                                  # 더 친화적
```

자세히 → [[../memory/allocators]]

---

## 12. /proc/pressure (PSI)

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
# some avg10=0.50 avg60=1.00 avg300=2.00 total=...
```

자세히 → [[../memory/swap-oom#10-psi--pressure-stall-information]]

---

## 13. /proc/$PID/oom_score / oom_score_adj

```bash
cat /proc/$PID/oom_score          # 0-1000
cat /proc/$PID/oom_score_adj       # -1000 to 1000

echo -500 > /proc/$PID/oom_score_adj
```

자세히 → [[../memory/swap-oom#6-oom-score]]

---

## 14. /sys/fs/cgroup — cgroup v2

자세히 → [[../virtualization/cgroups]]

```bash
ls /sys/fs/cgroup/
# cgroup.controllers  cpu.max  memory.max  io.max
# system.slice/  user.slice/

cat /sys/fs/cgroup/system.slice/nginx.service/cpu.stat
```

---

## 15. /proc 의 race

`/proc/$PID/...` read 중 process 죽으면 ENOENT.
파일이 atomic 한 snapshot 아님 (값이 변하는 중 read 가능).

→ monitoring 도구는 best-effort.

---

## 16. 응용 시나리오

### 16.1 메모리 사용 디버그

```bash
sort -k2 -n /proc/*/status 2>/dev/null | grep VmRSS | tail
# RSS 큰 process 찾기

cat /proc/$PID/smaps | grep ^Rss | awk '{s+=$2} END {print s}'
```

### 16.2 hang process 디버그

```bash
sudo cat /proc/$PID/stack             # kernel stack
sudo cat /proc/$PID/wchan              # 어디서 대기
```

### 16.3 sysctl 영구 변경

```bash
# /etc/sysctl.d/90-mine.conf
sudo sysctl --system
```

---

## 17. 함정

### 17.1 /proc 의 capacity 가정
process 100K → /proc 의 stat 도 100K → ps 가 느림.

### 17.2 sysctl 영구화 누락
재부팅 시 reset.

### 17.3 /sys 의 write 권한
대부분 root. sudo + 정확한 값.

### 17.4 /proc/$PID 의 race
read 중 process 죽음 → ENOENT.

### 17.5 cgroup v1 / v2 mount 위치
```
v1: /sys/fs/cgroup/<ctrl>/
v2: /sys/fs/cgroup/
```

### 17.6 namespace 안에서 본 /proc
host PID 와 다른 PID.

### 17.7 dmesg_restrict
일반 user 가 dmesg 못 봄 (보안). root 또는 cap.

### 17.8 cpu.max 의 quota / period
공식: `quota period` (μs). `max` 면 무제한.

---

## 18. 학습 자료

- `man 5 proc`
- `Documentation/filesystems/proc.rst`
- `Documentation/admin-guide/sysctl/`
- `Documentation/ABI/testing/sysfs-*`

---

## 19. 관련

- [[monitoring]]
- [[tuning]]
- [[../process/pcb]]
- [[linux]] — Linux hub
