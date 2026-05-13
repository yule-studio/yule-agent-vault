---
title: "Linux 모니터링 — top / vmstat / iostat / pidstat / sar"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:45:00+09:00
tags:
  - operating-system
  - linux
  - monitoring
---

# Linux 모니터링

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 도구 + USE 방법론 |

**[[linux|↑ Linux hub]]**

---

## 1. 시작 — uptime / w

```bash
uptime
# 16:00:00 up 100 days, 8 users, load average: 0.5, 1.2, 1.0

w
# 사용자 / TTY / from / login / idle / what
```

load average = 1, 5, 15 분 — runnable + uninterruptible task 평균. 코어 수와 비교.

---

## 2. CPU — top / htop

```bash
top
# - load
# - %us %sy %ni %id %wa %hi %si %st
# - process 별 CPU / MEM

htop                                    # 더 예쁜 / 색
btop                                    # 가장 예쁨
```

### 2.1 top 의 CPU 컬럼

| 약자 | 의미 |
| --- | --- |
| `us` | user |
| `sy` | system (kernel) |
| `ni` | nice |
| `id` | idle |
| `wa` | I/O wait |
| `hi` | hardware interrupt |
| `si` | software interrupt |
| `st` | stolen (VM 의 hypervisor 가 가져감) |

---

## 3. vmstat — 시스템 전체

```bash
vmstat 1
# procs ---memory---  ---swap-- ---io--- -system-- -----cpu-----
#  r  b  swpd free buff cache  si so   bi  bo   in   cs  us sy id wa st
#  1  0    0  500M 100M  2G    0  0   100  50  1000 500  10 5 80  5  0
```

| 컬럼 | 의미 |
| --- | --- |
| `r` | runnable (running + ready) |
| `b` | blocked (D state) |
| `swpd` | swap used |
| `free / buff / cache` | RAM |
| `si / so` | swap in/out |
| `bi / bo` | block in/out (KB/s) |
| `in / cs` | interrupt / context switch / s |
| `us / sy / id / wa / st` | CPU |

`r > 코어수` = 부하 ↑.

---

## 4. iostat — 디스크

```bash
iostat -x 1
# Device   r/s    w/s    rkB/s   wkB/s   await  svctm  %util
# sda      100    50     5000    2000    5      1      90.0
```

| 컬럼 | 의미 |
| --- | --- |
| `r/s w/s` | IOPS |
| `rkB/s wkB/s` | bandwidth |
| `r_await` | read latency (ms) |
| `w_await` | write latency |
| `aqu-sz` | queue 길이 |
| `%util` | 디스크 busy (saturating 가까울수록) |

`%util 100` = busy. NVMe 는 더 많은 queue → `%util` 의미 ↓ (queue 깊이 봐야).

---

## 5. pidstat — 프로세스 별

```bash
pidstat 1                              # CPU
pidstat -r 1                            # 메모리
pidstat -d 1                            # I/O
pidstat -w 1                            # context switch
pidstat -t -p $PID                      # 스레드
pidstat -p $PID 1                       # 특정 process
```

---

## 6. free / /proc/meminfo

```bash
free -h
#                total   used   free   shared  buff/cache   available
# Mem:           16Gi    4Gi    1Gi    100Mi   11Gi         11Gi
# Swap:          8Gi     0B     8Gi

# 핵심: available — 실제 가용 (buff/cache 포함)
```

```bash
cat /proc/meminfo                       # 자세히
```

자세히 → [[../memory/virtual-memory#13-procmeminfo]]

---

## 7. sar — 누적 / 과거

```bash
sudo apt install sysstat
sudo systemctl enable --now sysstat

sar -u 1                                # CPU
sar -r 1                                # 메모리
sar -d 1                                # 디스크
sar -n DEV 1                            # 네트워크
sar -n TCP 1                            # TCP
sar -q                                  # load
sar -W                                  # swap
sar -B                                  # page faults

# 과거 데이터
sar -u -f /var/log/sysstat/sa01         # 1일자
sar -u -s 14:00:00 -e 16:00:00
```

cron 으로 매 10 분 sa-file 저장 → 과거 분석 가능.

---

## 8. iotop — I/O top

```bash
sudo iotop
sudo iotop -o                            # active only
sudo iotop -P                            # process 별
```

---

## 9. mpstat — CPU 별

```bash
mpstat -P ALL 1
# 모든 코어 별 us/sy/id/wa
```

특정 코어 hotspot 발견.

---

## 10. nethogs / iftop — 네트워크

```bash
sudo nethogs                             # process 별 bandwidth
sudo iftop                               # 연결 별
sudo tcpdump                             # packet
sudo nload                               # interface 별

# 누적
cat /proc/net/dev
```

---

## 11. /proc/$PID/...

```bash
cat /proc/$PID/status                    # 메모리 / state
cat /proc/$PID/stat                      # raw (top 이 파싱)
cat /proc/$PID/io                        # I/O
cat /proc/$PID/limits
cat /proc/$PID/maps                      # 메모리 맵
cat /proc/$PID/sched
ls /proc/$PID/fd/                         # 열린 FD
```

자세히 → [[proc-sys]]

---

## 12. 시스템 부하 분석 — USE method

Brendan Gregg 의 USE — 모든 자원에 대해 3 가지:

| | 정의 | 도구 |
| --- | --- | --- |
| **U**tilization | 자원 사용률 (%) | top, iostat, sar |
| **S**aturation | 대기 queue / 부하 초과 | vmstat r, iostat aqu-sz |
| **E**rrors | 에러 / drop | dmesg, sar, /proc |

→ 어디가 bottleneck 인지 빠른 발견.

---

## 13. RED method — 응용

| | |
| --- | --- |
| **R**ate | 요청 수 / s |
| **E**rrors | 에러 수 |
| **D**uration | 응답 시간 |

Prometheus + Grafana 대시보드의 표준.

---

## 14. 4 Golden Signals (Google SRE)

1. Latency
2. Traffic (rate)
3. Errors
4. Saturation

→ 응용 + 인프라 모두.

---

## 15. PSI — Pressure Stall Info

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
# some avg10=... avg60=... avg300=... total=...
# full avg10=...
```

`some` 임의 task stall, `full` 모든 task stall. SLO 추세.

---

## 16. 외부 / 시각화

| 도구 | 의미 |
| --- | --- |
| **Prometheus** + **node_exporter** | 시계열 / 표준 |
| **Grafana** | 대시보드 |
| **Datadog / NewRelic / Dynatrace** | APM SaaS |
| **Zabbix** | 옛 운영 |
| **Netdata** | 한 노드 실시간 web |
| **Cockpit** | 한 노드 web UI |
| **Glances** | terminal 종합 |
| **PCP (Performance Co-Pilot)** | 분산 |

---

## 17. log + journal

```bash
journalctl -p err
journalctl --since "1 hour ago"
journalctl -k                            # kernel
dmesg -T -w
```

자세히 → [[logging]]

---

## 18. eBPF observability

```bash
sudo bpftrace -e '...'
bcc-tools:
  execsnoop / opensnoop / biolatency / tcptop / ...
```

자세히 → [[ebpf]]

---

## 19. perf

```bash
sudo perf top
sudo perf stat -a sleep 5
sudo perf record -F 99 -a -g -- sleep 30
sudo perf report
```

자세히 → [[strace-perf]]

---

## 20. 함정

### 20.1 load average vs CPU
load 는 R + D. high load 가 무조건 CPU busy 아님.

### 20.2 %util 과 NVMe
NVMe multi-queue → %util 100% 도 여유 있음.

### 20.3 buff/cache 가 사용
"free 가 0" 보고 OOM 가정 X. `available` 봐야.

### 20.4 wa (I/O wait)
실제 I/O 보다 응용 sleep 비율 — 정확 측정은 PSI.

### 20.5 sar 의 paste 단위
다른 sar 명령 마다 다른 단위 (KB/s vs MB/s).

### 20.6 top 의 평균
첫 sample 은 부팅 후 평균. 두 번째 부터 의미.

### 20.7 hard 자원 미고려
RAM / 디스크 외 — file descriptor / inotify watch / TCP port 도 자원.

### 20.8 monitoring 자체의 overhead
strace / 큰 bpftrace 가 응용 영향.

---

## 21. 학습 자료

- **Systems Performance** — Brendan Gregg
- **brendangregg.com** — flame graph / USE
- **The Linux Programming Interface** Ch. 28 (proc)
- **Site Reliability Engineering** — Google

---

## 22. 관련

- [[logging]]
- [[strace-perf]]
- [[ebpf]]
- [[tuning]]
- [[linux]] — Linux hub
