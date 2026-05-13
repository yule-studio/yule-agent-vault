---
title: "Process (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:00:00+09:00
tags:
  - operating-system
  - process
  - hub
---

# Process (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 5 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄 정의

**실행 중인 프로그램** + 자신만의 가상 주소 공간 + 자원 (FD, signal mask, ...).
프로그램은 디스크의 파일, 프로세스는 RAM 의 실행 인스턴스.

---

## 2. 프로세스의 구성

```
┌────────────────────┐
│  Code (Text)       │  ← 실행 코드 (read-only)
├────────────────────┤
│  Data              │  ← 전역 변수
├────────────────────┤
│  Heap              │  ← malloc / new (위로 자람)
├────────────────────┤
│  ...               │
├────────────────────┤
│  Stack             │  ← 함수 호출 (아래로 자람)
└────────────────────┘
+ PCB (Process Control Block): PID, FD table, signal, mm, sched 정보
```

---

## 3. PCB (Process Control Block) — Linux `task_struct`

Linux 의 한 프로세스 = `task_struct` 구조체 인스턴스.
주요 필드:
- pid, tgid, parent, children
- state (TASK_RUNNING / INTERRUPTIBLE / ...)
- mm (memory map), files (FD table)
- signal handlers
- sched_entity (CFS)

자세히 → [[pcb]]

---

## 4. 프로세스 상태

```
NEW → READY ⇄ RUNNING → TERMINATED
        ↑       ↓
        └ BLOCKED ←
```

Linux:
| 상태 | 의미 |
| --- | --- |
| `R` (Running / Runnable) | CPU 점유 또는 ready queue |
| `S` (Interruptible Sleep) | 시그널로 깨움 가능 |
| `D` (Uninterruptible Sleep) | I/O 등 시그널 X — 위험 |
| `T` (Stopped) | SIGSTOP |
| `Z` (Zombie) | 종료됐지만 부모가 wait() 안 함 |

자세히 → [[states]]

---

## 5. 프로세스 생명주기 — fork / exec / wait / exit

```c
pid_t pid = fork();
if (pid == 0) {
    // child
    execvp("ls", args);
    _exit(1);          // 실패 시
} else if (pid > 0) {
    // parent
    int status;
    waitpid(pid, &status, 0);
}
```

자세히 → [[fork-exec]]

---

## 6. 시그널

프로세스 간 비동기 알림.

```c
signal(SIGINT, handler);
kill(pid, SIGTERM);
raise(SIGUSR1);
```

| 시그널 | 의미 |
| --- | --- |
| SIGINT | Ctrl+C |
| SIGTERM | 정상 종료 요청 |
| SIGKILL | 강제 종료 (catch X) |
| SIGSTOP | 일시 정지 (catch X) |
| SIGSEGV | 잘못된 메모리 |
| SIGCHLD | 자식 상태 변경 |
| SIGUSR1/2 | 사용자 정의 |

자세히 → [[signals]]

---

## 7. Zombie / Orphan

| 상태 | 부모 | 자식 |
| --- | --- | --- |
| Zombie | 살아있음, wait X | exit 했지만 PCB 남음 |
| Orphan | 죽음 | 살아있음 → init (PID 1) 양도 |

자세히 → [[orphan-zombie]]

---

## 8. 프로세스 vs 스레드

| 항목 | Process | Thread |
| --- | --- | --- |
| 주소 공간 | 독립 | 공유 |
| 생성 비용 | 큼 (fork) | 작음 (pthread_create) |
| 컨텍스트 스위치 | 무거움 (TLB flush) | 가벼움 |
| 통신 | IPC | 메모리 직접 |
| 격리 | 강 | 약 |

자세히 → [[../threads/threads]]

---

## 9. Linux 에서 보기

```bash
ps aux                          # 모든 프로세스
ps -ef --forest                  # 트리
pstree -p
top / htop                       # 실시간
pgrep nginx
pidof nginx

cat /proc/$PID/status            # task_struct 일부
cat /proc/$PID/maps              # 메모리 맵
cat /proc/$PID/fd/               # 열린 파일
ls -l /proc/$PID/exe             # 실행 파일

# 시그널
kill -TERM $PID
kill -9 $PID                     # SIGKILL (최후 수단)
killall nginx
```

---

## 10. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[states]] | 프로세스 상태 + Linux state codes |
| [[pcb]] | PCB / `task_struct` 구조 |
| [[fork-exec]] | fork / exec / CoW / vfork |
| [[signals]] | 시그널 / 핸들러 / async-signal-safe |
| [[orphan-zombie]] | Zombie / Orphan / SIGCHLD |

---

## 11. 함정

### 함정 1 — fork() 후 멀티스레드 호출
fork 는 호출 스레드만 복사. 다른 스레드가 잠고 있던 락 = 자식에서 영원히 lock.

### 함정 2 — Zombie 누적
부모가 wait() 안 함 → PCB 누적. `signal(SIGCHLD, SIG_IGN)` 또는 wait.

### 함정 3 — SIGKILL 남용
정리 코드 (atexit, dtor) 실행 X. SIGTERM 후 grace period.

### 함정 4 — D state 프로세스
Uninterruptible — kill 도 안 됨. 디스크 / NFS / 커널 버그 의심.

### 함정 5 — 큰 프로세스의 fork
RAM 동등 복사 환상 (CoW 로 lazy). 그러나 page table 자체 복제 비용 ↑.

---

## 12. 관련

- [[../threads/threads]] — 경량 실행 단위
- [[../scheduling/scheduling]] — CPU 분배
- [[../ipc/ipc]] — 프로세스 간 통신
- [[../operating-system|↑ OS hub]]
