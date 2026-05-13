---
title: "Swap / OOM Killer"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:20:00+09:00
tags:
  - operating-system
  - memory
  - swap
  - oom
---

# Swap / OOM Killer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | swap / overcommit / OOM |

**[[memory|↑ Memory hub]]**

---

## 1. Swap

RAM 부족 → 사용 안 하는 페이지를 디스크의 **swap 영역** 으로 옮김. 나중에 필요하면 다시 RAM 으로.

```
RAM 16 GB + swap 16 GB = 32 GB 가용 (느림)
```

---

## 2. swap 설정

```bash
# swap 파일 생성
sudo dd if=/dev/zero of=/swapfile bs=1M count=16384
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapoff -a

# 영구화 /etc/fstab
/swapfile  none  swap  sw  0 0

# 파티션도 mkswap + swapon
```

```bash
swapon --show
free -h
```

---

## 3. swappiness

```bash
cat /proc/sys/vm/swappiness
sysctl vm.swappiness=10
```

| 값 | 동작 |
| --- | --- |
| 0 | swap 거의 X (RAM 비울 때 file cache 우선 evict) |
| 60 (기본) | 균형 |
| 100 | anon page 도 적극 swap |

DB / latency 민감 = 1 ~ 10.

---

## 4. Overcommit

Linux 는 기본 **overcommit** — malloc 이 가상만 잡고 성공 → 실제 접근 시 page fault.

```bash
cat /proc/sys/vm/overcommit_memory
# 0 (기본) — heuristic
# 1         — 무조건 OK (BGSAVE fork 안전)
# 2         — strict (실제 가용만)

cat /proc/sys/vm/overcommit_ratio
# 50% (기본) + swap 만큼
```

---

## 5. OOM (Out Of Memory)

```
실제 RAM + swap 모두 부족 → 새 page fault 실패
→ OOM Killer 발동 → 어떤 프로세스 SIGKILL
```

---

## 6. OOM Score

```bash
cat /proc/$PID/oom_score
cat /proc/$PID/oom_score_adj
```

- 0 ~ 1000
- 높을수록 먼저 죽음
- `oom_score_adj` (-1000 ~ +1000) 으로 사용자 조정
  - -1000 = 절대 안 죽음
  - +1000 = 가장 먼저

```bash
echo -500 > /proc/$PID/oom_score_adj
```

systemd:
```ini
[Service]
OOMScoreAdjust=-500
```

---

## 7. OOM 로그

```bash
dmesg | grep -i 'killed process'
journalctl -k --grep oom

# 시스템 정보 (메모리 / task 목록 / score)
```

```
Out of memory: Killed process 1234 (mysqld)
total-vm:8000000kB anon-rss:6000000kB file-rss:1000kB
oom_score_adj:0
```

---

## 8. OOM 회피

1. **메모리 측정 / leak 잡기** — heap profile / `pmap`
2. **swap 추가** — 단, latency 폭증 위험
3. **cgroup memory.max** — 한 그룹 안에서만 OOM
4. **oom_score_adj** 으로 보호할 프로세스 명시
5. **earlyoom / oomd** — 사용자 공간 OOM 매니저

---

## 9. earlyoom / systemd-oomd

전통 커널 OOM Killer = 너무 늦게 동작 (이미 thrashing 중) → freeze 길어짐.

**earlyoom / systemd-oomd** = PSI (Pressure Stall Info) 모니터 → 일찍 죽임.

```bash
sudo systemctl enable --now systemd-oomd
```

Fedora 등 기본 활성.

---

## 10. PSI — Pressure Stall Information

```bash
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

- `some` — 어떤 task 가 메모리 stall
- `full` — 모든 task stall (시스템 정지)

oomd 가 이 값으로 OOM 시점 판단.

---

## 11. cgroup memory.max

```bash
echo 4G > /sys/fs/cgroup/myapp/memory.max
```

cgroup 안의 RSS + page cache 합이 4 GB 초과 → 그 cgroup 안의 task OOM.

→ 시스템 전체 OOM 보호.

Docker `--memory=4g` / K8s `limits.memory: 4Gi` 가 이걸 사용.

---

## 12. zswap / zram

```
zswap: swap 으로 가기 전 메모리에서 압축 — 빠른 swap-like
zram:  RAM 의 압축 블록 디바이스를 swap 으로 사용 — 디스크 X
```

```bash
modprobe zram
echo 4G > /sys/block/zram0/disksize
mkswap /dev/zram0
swapon /dev/zram0
```

Chrome OS / Android / Fedora 기본 사용.

---

## 13. 응용 시나리오

### 13.1 DB 서버
- swap = 0 또는 매우 작게 (~1 GB, 안전망)
- swappiness = 1
- mlock 으로 buffer pool 보호
- cgroup memory.max

### 13.2 데스크탑
- swap 8-16 GB
- zram 으로 swap 가속
- earlyoom / oomd

### 13.3 K8s
- swap 비활성 (kubelet 권장)
- 1.22+ swap 지원하지만 권장 X
- limit / request 정밀

---

## 14. swap 없을 때

```
RAM 부족 → 즉시 OOM (지연 X)
```

- 빠른 죽음 (lockup 안 함)
- 데이터 보존 ↑
- vs swap 있을 때: 천천히 죽음 + thrashing 가능

DB 등 latency 민감은 swap 거의 X (안전망 1-2 GB).

---

## 15. 측정

```bash
free -h
# 핵심: available (실제 가용)

vmstat 1
# si/so = swap in/out per second — 활동 중인지

sar -W 1
# pswpin/s pswpout/s

cat /proc/meminfo
# MemAvailable, SwapFree, SwapTotal
```

---

## 16. 함정

### 16.1 swappiness 너무 큼
DB buffer pool swap → 성능 X.

### 16.2 swap 없음 + overcommit
malloc 성공 후 page fault 시 OOM.

### 16.3 OOM Killer 가 wrong process kill
oom_score_adj 로 명시.

### 16.4 cgroup memory.max 없이 운영
한 컨테이너 leak 으로 호스트 전체 OOM.

### 16.5 thrashing
swap 폭증 → 응답 X. earlyoom / 작업 축소.

### 16.6 K8s swap 활성
1.22 부터 지원이지만 latency 예측 X. 보통 비활성.

### 16.7 zram + swap 동시
복잡 — 측정 후 결정.

### 16.8 OOM 로그 분석 누락
dmesg / journalctl 확인.

---

## 17. 학습 자료

- **OSTEP** Ch. 21-22
- `Documentation/admin-guide/mm/`
- **Brendan Gregg — Memory** 시리즈
- **systemd-oomd man page**

---

## 18. 관련

- [[page-fault]]
- [[page-replacement]]
- [[../virtualization/cgroups]]
- [[memory]] — Memory hub
