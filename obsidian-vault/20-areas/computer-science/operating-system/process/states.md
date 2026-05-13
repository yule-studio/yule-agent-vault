---
title: "프로세스 상태 — Linux State Codes"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:05:00+09:00
tags:
  - operating-system
  - process
  - state
---

# 프로세스 상태

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 상태 다이어그램 + Linux codes |

**[[process|↑ Process hub]]**

---

## 1. 기본 5 상태 모델

```
        ┌─────┐
        │ NEW │
        └──┬──┘
           ↓ (admission)
        ┌──────┐
   ┌────│ READY│←──────┐
   │    └──┬───┘       │
   │       ↓ (dispatch)│
   │    ┌──────────┐   │ wake up
   │    │ RUNNING  │   │
   │    └──┬──┬────┘   │
   │ exit  │  │ block  │
   │       ↓  ↓        │
   ↓   ┌────┐ ┌────────┴───┐
 TERMI │exit│ │  BLOCKED   │
 NATED └────┘ │ (waiting)  │
              └────────────┘
```

- **NEW** — 생성 중 (자원 할당)
- **READY** — 실행 준비, CPU 기다림
- **RUNNING** — CPU 점유
- **BLOCKED / WAITING** — I/O / 락 등 대기
- **TERMINATED** — 종료

---

## 2. Linux 상태 코드 (`ps`, `top`)

```bash
ps -eo pid,state,comm | head
```

| Code | 의미 |
| --- | --- |
| `R` | Running / Runnable (CPU 점유 또는 ready) |
| `S` | Interruptible Sleep — 시그널로 깨움 가능 |
| `D` | Uninterruptible Sleep — 보통 디스크 I/O |
| `T` | Stopped (SIGSTOP, ptrace) |
| `t` | Stopped by debugger |
| `Z` | Zombie (defunct) |
| `X` | Dead (보이지 않음) |
| `I` | Idle (커널 스레드) |
| `<` | High priority |
| `N` | Low priority (niced) |
| `L` | Locked pages in memory |
| `s` | Session leader |
| `l` | Multi-threaded |
| `+` | Foreground process group |

### 2.1 예

```
$ ps aux
USER   PID  STAT  COMMAND
root     1  Ss   /sbin/init
www    234  Rsl  /usr/sbin/nginx
www    235  S    nginx: worker
db     500  D    postgres: writer
me     999  Z    [chrome] <defunct>
```

---

## 3. R — Running / Runnable

CPU 위에 있거나 ready queue 에서 대기. `top` 의 `R` 카운트가 많으면 = CPU bound.

```bash
vmstat 1
# r 컬럼 = R 상태 프로세스 수
# r > CPU 코어 수 → 부하 ↑
```

---

## 4. S — Interruptible Sleep

대부분의 idle 프로세스. socket / FIFO / select / sleep 등.

- 시그널 받으면 즉시 깨어남
- 가장 흔한 상태

---

## 5. D — Uninterruptible Sleep ⚠️

```
프로세스가 시그널 X — kill -9 도 안 먹힘
```

원인:
- 디스크 I/O (보통 ms 단위로 지나감)
- NFS hang
- 커널 드라이버 버그
- 잘못된 mount

### 5.1 진단

```bash
# D state 인 프로세스 wchan 확인
ps -eo pid,state,wchan,comm | awk '$2=="D"'

# kernel stack
cat /proc/$PID/stack
```

### 5.2 해결
- NFS / 디스크 / 드라이버 확인
- 리부트가 사실상 유일한 해결책 (D 가 오래 가면)

---

## 6. T — Stopped

```bash
kill -STOP $PID                  # SIGSTOP
kill -CONT $PID                  # 재개

# 또는 fg/bg
Ctrl-Z                           # 셸에서 SIGTSTP → T
bg
fg
```

debugger (gdb) 도 T 로 만듦.

---

## 7. Z — Zombie

```
자식이 exit → PCB 만 남아 부모가 wait() 할 때까지 보관
부모가 wait X → 누적
```

```bash
ps aux | awk '$8=="Z" {print}'
# 또는
ps -eo pid,ppid,state,comm | grep Z
```

자세히 → [[orphan-zombie]]

---

## 8. I — Idle (Kernel)

커널 스레드 (`[kworker]`, `[ksoftirqd]`) 의 idle 상태. R/S/D 와 다른 카운트.

---

## 9. 상태 전이

| 전이 | 원인 |
| --- | --- |
| READY → RUNNING | 스케줄러 dispatch |
| RUNNING → READY | timeslice 만료, preempt |
| RUNNING → BLOCKED | I/O, lock, sleep |
| BLOCKED → READY | I/O 완료, lock 해제 |
| RUNNING → TERMINATED | exit, signal |

---

## 10. 우선순위 (Linux)

```bash
nice -n 10 ./script.sh             # nice 값 +10 (낮은 우선)
renice -n -5 -p $PID                # 더 높은 우선 (root)

# 실시간
chrt -f 50 ./realtime               # SCHED_FIFO priority 50
chrt -r 50 ./realtime               # SCHED_RR
```

| 정책 | 의미 |
| --- | --- |
| `SCHED_OTHER` (CFS, 기본) | 일반 |
| `SCHED_BATCH` | 배치 워크로드 |
| `SCHED_IDLE` | 가장 낮은 |
| `SCHED_FIFO` | 실시간, FIFO |
| `SCHED_RR` | 실시간, Round Robin |
| `SCHED_DEADLINE` | EDF |

자세히 → [[../scheduling/scheduling]]

---

## 11. cgroup state

cgroup v2 로 프로세스 / 스레드 그룹의 CPU / 메모리 / I/O 제한.

```bash
cat /sys/fs/cgroup/<group>/cgroup.procs
cat /sys/fs/cgroup/<group>/cpu.stat
```

자세히 → [[../virtualization/cgroups]]

---

## 12. 진단 도구

```bash
ps aux
top
htop                                # 컬러풀
btop                                # 더 예쁨
glances

pidstat 1                           # 프로세스별 CPU
pidstat -d 1                        # I/O
pidstat -r 1                        # 메모리

# 상태별 카운트
ps -eo state | sort | uniq -c
```

---

## 13. 함정

### 함정 1 — `D` 상태 무시
짧은 D 는 정상. 지속 D 는 비상.

### 함정 2 — `Z` 누적
부모가 SIGCHLD 무시 / wait 안 함 → 좀비 누적 → PID 고갈.

### 함정 3 — `R` 많음 = OK 아님
R > CPU 코어 = run queue 길어짐 = latency ↑.

### 함정 4 — SIGSTOP 후 잊음
T 상태 프로세스가 자원 잡고 있음.

### 함정 5 — nice 값과 우선순위 혼동
nice 는 -20 ~ +19, 낮을수록 우선. priority 와 다름.

---

## 14. 관련

- [[fork-exec]] — 생성
- [[signals]] — 상태 변화 알림
- [[orphan-zombie]] — Z 상세
- [[../scheduling/scheduling]] — READY → RUNNING
- [[process]] — Process hub
