---
title: "Thrashing — 시스템 압박"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:35:00+09:00
tags:
  - operating-system
  - topic
  - thrashing
---

# Thrashing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 메모리 / 캐시 thrashing |

**[[topics|↑ Topics hub]]**

---

## 1. 한 줄

CPU 의 대부분이 **유용한 일** 보다 **자원 부족 해소** 에 쓰이는 상태.

```
정상:   CPU 90% useful work + 10% overhead
thrash: CPU 10% useful + 90% overhead (page swap / cache miss / TLB miss)
```

→ 응답 X. 시스템 freeze 가까운 상태.

---

## 2. Page Thrashing (가장 흔함)

```
RSS > 사용 가능 RAM
→ 페이지 교체 폭증
→ 디스크 swap I/O 가 99%
→ 응용 stall
```

증상:
- 응답 X
- CPU `wa` 매우 높음
- swap I/O 폭증 (vmstat si/so)
- load average 폭증
- ssh 도 느림

```bash
vmstat 1
# si so 가 큰 값 (수십 MB/s)

free -h
# used 가 거의 RAM, swap 도 거의 차거나 활발

sar -W 1
# pswpin/s pswpout/s 큰 값
```

---

## 3. 왜 발생

| 원인 | 의미 |
| --- | --- |
| **메모리 leak** | RSS 점진 ↑ |
| **fork + COW + write** | Redis BGSAVE 함정 |
| **갑작스러운 workload** | batch / 분석 |
| **memory limit 잘못** | container / VM |
| **swap on + 작은 RAM** | swap 의 dark side |
| **cgroup overcommit** | 여러 container 합 > 노드 |

---

## 4. 회복 전략

### 4.1 응급
```bash
# OOM Killer 강제 (큰 process 죽임)
sudo kill -9 $(ps aux --sort=-rss | head -2 | tail -1 | awk '{print $2}')

# swap 끄기 (회복 가속)
sudo swapoff -a
# → OOM 발생, 일부 process 죽지만 시스템 회복
```

### 4.2 장기
- 메모리 leak 잡기
- swap 끄거나 작게 + swappiness=1
- earlyoom / systemd-oomd
- cgroup memory.high 활용
- 정기 restart

자세히 → [[../memory/swap-oom]], [[oom-scenarios]]

---

## 5. earlyoom / systemd-oomd

```
일반 OOM Killer = 너무 늦게 (이미 thrashing 중)
earlyoom / oomd = PSI 모니터 → 일찍 죽임 → freeze 단축
```

```bash
sudo systemctl enable --now systemd-oomd
```

Fedora / Ubuntu 23.04+ 기본 활성.

---

## 6. PSI — Pressure Stall Information

```bash
cat /proc/pressure/memory
# some avg10=85.50 avg60=70.0 avg300=50.0 total=...
# full avg10=...
```

`some > 60` 정도면 thrashing 가까움.
`full > 0` 이면 시스템 정지.

자세히 → [[../memory/swap-oom#10-psi--pressure-stall-information]]

---

## 7. Cache Thrashing (CPU)

```
working set > cache size
→ cache miss 폭증
→ 메모리 latency × 100
```

증상:
- IPC 매우 낮음 (< 0.5)
- cache-miss rate 높음

진단:
```bash
sudo perf stat -e cycles,instructions,cache-references,cache-misses ./app
```

해결:
- 작업 set 줄임 (loop 안 더 적은 데이터)
- cache-friendly 자료구조 (linear)
- false sharing 회피
- huge page

자세히 → [[cache-locality]], [[../memory/huge-pages]]

---

## 8. TLB Thrashing

```
working set 의 페이지 > TLB coverage
→ TLB miss 폭증
→ page walk 비용 ↑
```

해결: **Huge Page** — 4 KB × 64 entry → 2 MB × 64 entry = 128 MB coverage.

자세히 → [[../memory/tlb]]

---

## 9. Context Switch Thrashing

```
너무 많은 스레드 / blocking → context switch 폭증
→ context switch 자체 가 CPU 의 큰 부분
```

증상:
- vmstat 의 cs 매우 큼 (10 K+/s)
- IPC 낮음
- 응용 latency 폭증

해결:
- 스레드 수 줄임 (= 코어 수 기준)
- async / event loop
- pin to CPU

자세히 → [[../scheduling/context-switch]]

---

## 10. Lock Thrashing

```
많은 스레드 + 같은 mutex → 락 경합 폭증
→ CPU 의 대부분이 spin / wait
```

진단:
```bash
sudo perf record -e sched:sched_switch -a -g sleep 30
sudo perf report
# 또는
sudo bpftrace -e 'tracepoint:sched:sched_switch /args->prev_state == TASK_INTERRUPTIBLE/ { ... }'
```

해결:
- lock contention 진단 → 락 범위 축소 / 분할
- lock-free (atomic / RCU)
- finer-grained lock
- read-write lock

자세히 → [[../synchronization/synchronization]]

---

## 11. Disk Thrashing

```
random I/O 폭증 (특히 HDD)
→ seek time 90% 의 시간
→ throughput 폭락
```

해결:
- SSD / NVMe
- batch / sequential
- write-back cache
- 응용의 access pattern 조정

---

## 12. Network Thrashing

```
packet 폭증 → buffer overflow → retransmit → 폭증
```

해결:
- queue size 조정
- congestion control (BBR)
- rate limiting
- traffic shaping

---

## 13. GC Thrashing (JVM)

```
heap 가까이 가득 → GC 자주 → CPU 의 대부분이 GC
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

해결:
- heap 늘림
- GC tuning (G1 / ZGC / Shenandoah)
- 메모리 leak 잡기

---

## 14. 정상 시스템의 지표

| 지표 | 정상 |
| --- | --- |
| CPU `wa` | < 5% |
| Swap I/O (si/so) | 거의 0 |
| Load average | < CPU 수 |
| cs / s | < 10K |
| cache miss rate | < 5% |
| IPC | > 1.0 |
| PSI memory | < 10 |

---

## 15. 함정

### 15.1 swap on 의 함정
RAM 부족 시 swap → thrash. swap 작게 / 끄기.

### 15.2 freeze 후 ssh 도 안 됨
console 만 가능 — sysrq + Magic SysRq key:
```
Alt + SysRq + F   = force OOM Killer
Alt + SysRq + B   = reboot
```

### 15.3 OOM Killer 가 늦음
earlyoom / oomd.

### 15.4 K8s overcommit
node 의 합 limit > 노드 RAM. requests 으로 보호.

### 15.5 cgroup memory.max 만
hard cliff. memory.high 으로 부드러운 reclaim.

### 15.6 thrashing 후 RSS 가 줄지 않음
memory leak 의 표시.

---

## 16. 학습 자료

- **Operating Systems Concepts** (Silberschatz) — Thrashing
- **The Linux Programming Interface** Ch. 49
- **Brendan Gregg — Memory** 시리즈
- **PSI documentation**

---

## 17. 관련

- [[../memory/swap-oom]]
- [[../memory/page-replacement]]
- [[oom-scenarios]]
- [[cache-locality]]
- [[topics]] — Topics hub
