---
title: "Linux 튜닝 — sysctl / ulimit / scheduler"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:20:00+09:00
tags:
  - operating-system
  - linux
  - tuning
---

# Linux 튜닝

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 자주 쓰는 튜닝 모음 |

**[[linux|↑ Linux hub]]**

---

## 1. 원칙

1. **측정 후 튜닝** — guess work X
2. **한 번에 1 가지** 변경
3. baseline + 비교
4. 영구화 (`sysctl.d` / `systemd` / `fstab`)
5. 운영 적용 전 staging 검증

---

## 2. ulimit (per-process)

```bash
ulimit -a
ulimit -n 65535                      # file descriptor
ulimit -u 4096                        # process / thread
ulimit -s 8192                        # stack size (KB)
ulimit -c unlimited                   # core dump

# 영구
# /etc/security/limits.conf
*     soft  nofile  65535
*     hard  nofile  65535
*     soft  nproc   4096
*     hard  nproc   8192
```

systemd:
```ini
[Service]
LimitNOFILE=65535
LimitNPROC=4096
LimitCORE=infinity
```

서버 응용은 항상 nofile 증가.

---

## 3. sysctl — kernel 변수

```bash
sysctl -a                              # 모두
sysctl vm.swappiness
sudo sysctl vm.swappiness=10           # 임시
```

영구:
```bash
# /etc/sysctl.d/99-mine.conf
vm.swappiness=10
net.core.somaxconn=4096
fs.file-max=2097152

sudo sysctl --system                   # 적용
```

---

## 4. Network sysctl 핵심

```ini
# socket queue
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 8192

# TCP buffer
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# 빠른 재사용
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# keepalive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 3

# congestion control
net.ipv4.tcp_congestion_control = bbr

# port range
net.ipv4.ip_local_port_range = 1024 65535

# IPv4 forwarding (router / NAT)
net.ipv4.ip_forward = 1

# 보안
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
```

자세히 → [[networking]]

---

## 5. Memory sysctl

```ini
vm.swappiness = 1                      # DB / latency
vm.vfs_cache_pressure = 50
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10

vm.overcommit_memory = 1               # 또는 0 (heuristic)
vm.overcommit_ratio = 50

vm.max_map_count = 262144              # Elasticsearch / mmap
vm.min_free_kbytes = 65536
```

자세히 → [[../memory/swap-oom]], [[../memory/virtual-memory]]

---

## 6. File system sysctl

```ini
fs.file-max = 2097152
fs.nr_open = 1048576
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192
fs.aio-max-nr = 1048576
```

---

## 7. THP (Transparent Huge Page)

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

영구 (systemd):
```ini
# /etc/systemd/system/disable-thp.service
[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/enabled"
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
```

DB / Redis / MongoDB 권장.

자세히 → [[../memory/huge-pages]]

---

## 8. NUMA

```bash
numactl --hardware
numastat
numactl --interleave=all ./app
numactl --cpunodebind=0 --membind=0 ./app
```

자세히 → [[../memory/numa]]

---

## 9. CPU governor

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# 성능 우선
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 절전
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 영구
sudo apt install cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
```

서버 = performance. 노트북 = powersave / schedutil.

---

## 10. Disk scheduler

```bash
cat /sys/block/sda/queue/scheduler
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler        # NVMe
echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler      # SSD/HDD
echo bfq | sudo tee /sys/block/sda/queue/scheduler              # desktop
```

`/etc/udev/rules.d/60-scheduler.rules`:
```
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
```

---

## 11. Mount option

```
/etc/fstab:
/dev/sda1 / ext4 defaults,noatime,nodiratime,discard 0 1

# noatime — atime 갱신 X
# discard — SSD TRIM
# data=writeback — 빠르지만 안전 ↓
```

자세히 → [[../filesystem/ext-xfs-zfs#14-mount-option-핵심]]

---

## 12. fs.aio-max-nr (DB)

```bash
sysctl fs.aio-max-nr=1048576
```

InnoDB / Oracle 의 AIO 한계.

---

## 13. RT / latency

```bash
# RT priority
chrt -f 50 ./rt-app

# CPU pinning
taskset -c 2,3 ./app

# core 격리 (boot param)
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3

# IRQ 분산
cat /proc/interrupts
echo 4 > /proc/irq/19/smp_affinity     # IRQ 19 to CPU 2
```

자세히 → [[../scheduling/realtime]]

---

## 14. Profile / Tuned

```bash
sudo apt install tuned
sudo systemctl enable --now tuned
sudo tuned-adm list
sudo tuned-adm profile throughput-performance
sudo tuned-adm profile latency-performance
sudo tuned-adm profile network-latency
```

profile 별 자동 sysctl + scheduler + CPU governor.

| Profile | 의미 |
| --- | --- |
| `balanced` | desktop 균형 |
| `throughput-performance` | 서버 처리량 |
| `latency-performance` | 낮은 latency |
| `network-latency` | network 우선 |
| `network-throughput` | network 처리량 |
| `virtual-guest` | VM 안 |
| `virtual-host` | hypervisor |

---

## 15. systemd 의 자원 제한

```ini
# /etc/systemd/system/myapp.service
[Service]
MemoryMax=4G
CPUQuota=200%
TasksMax=512
IOWeight=200
LimitNOFILE=65535
LimitNPROC=4096
OOMScoreAdjust=-500
Nice=-5
```

자세히 → [[systemd]], [[../virtualization/cgroups]]

---

## 16. K8s + 호스트 OS

```yaml
spec:
  template:
    spec:
      securityContext:
        sysctls:
        - name: net.ipv4.ip_local_port_range
          value: "1024 65535"
        - name: fs.inotify.max_user_instances
          value: "8192"
```

`safe` sysctl 만 — 다른 것은 kubelet 의 `--allowed-unsafe-sysctls`.

---

## 17. 자주 보는 워크로드 별 튜닝

### 17.1 웹 서버 (Nginx)

```ini
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_tw_reuse=1
fs.file-max=2097152
# nginx ulimit -n 65535
```

### 17.2 데이터베이스 (Postgres / MySQL)

```ini
vm.swappiness=1
vm.dirty_background_ratio=2
vm.dirty_ratio=5
vm.overcommit_memory=2 (또는 1)
vm.overcommit_ratio=80
# THP off
# Huge Page reserved
fs.aio-max-nr=1048576
```

### 17.3 Elasticsearch / 검색 엔진

```ini
vm.max_map_count=262144
vm.swappiness=1
# THP madvise
# nofile 65535
# memory lock unlimited
```

### 17.4 Kafka / Cassandra

```ini
vm.swappiness=1
vm.max_map_count=1048575
net.core.rmem_max=16777216
net.core.wmem_max=16777216
# disk I/O scheduler = none / mq-deadline
```

---

## 18. 디스크 / 파일 시스템

```bash
# 디스크 정보
sudo lsblk -f
sudo hdparm -I /dev/sda
sudo smartctl -a /dev/sda

# benchmark
sudo fio --name=write --size=1G --bs=4k --iodepth=32 --rw=randwrite

# TRIM
sudo fstrim -v /

# 정기 TRIM
sudo systemctl enable --now fstrim.timer
```

---

## 19. 함정

### 19.1 sysctl 영구화 누락
재부팅 시 reset.

### 19.2 한 번에 너무 많은 변경
원인 추적 어려움.

### 19.3 측정 없음
"빨라진 듯" — 실제 측정 필요.

### 19.4 ulimit nofile 부족
"Too many open files" — 너무 흔함.

### 19.5 THP enabled 의 latency spike
DB / Redis 권장 off.

### 19.6 vm.swappiness=0 (옛 의미)
지금은 0 이 매우 적극적 X — 1 권장.

### 19.7 governor=powersave on 서버
지속 부하 시 throttle 폭증.

### 19.8 mq-deadline on NVMe
NVMe = none. 옛 scheduler 가 오히려 손해.

### 19.9 nodatacow + Btrfs + DB
Btrfs 의 COW = DB 와 충돌. `chattr +C` 또는 다른 fs.

---

## 20. 학습 자료

- **Systems Performance** — Brendan Gregg
- **Red Hat / Oracle Tuning Guide**
- `Documentation/admin-guide/sysctl/`
- **tuned profile** 분석
- **AWS / GCP Performance Guides**

---

## 21. 관련

- [[monitoring]]
- [[proc-sys]]
- [[../memory/memory]]
- [[../scheduling/scheduling]]
- [[linux]] — Linux hub
