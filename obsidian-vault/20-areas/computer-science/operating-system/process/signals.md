---
title: "시그널 (Signal) — 비동기 알림"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:20:00+09:00
tags:
  - operating-system
  - process
  - signal
---

# 시그널 (Signal)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 시그널 / 핸들러 / async-signal-safe |

**[[process|↑ Process hub]]**

---

## 1. 한 줄

프로세스 / 스레드 사이의 **비동기 알림**. 1 비트 메시지 (시그널 번호 + siginfo 옵션).

---

## 2. 주요 시그널

| 시그널 | 번호 | 기본 | 의미 |
| --- | --- | --- | --- |
| `SIGHUP` | 1 | terminate | 터미널 끊김, 데몬 reload 관례 |
| `SIGINT` | 2 | terminate | Ctrl+C |
| `SIGQUIT` | 3 | core | Ctrl+\ |
| `SIGILL` | 4 | core | 잘못된 명령어 |
| `SIGTRAP` | 5 | core | breakpoint |
| `SIGABRT` | 6 | core | `abort()` |
| `SIGBUS` | 7 | core | 잘못된 메모리 정렬 / mmap |
| `SIGFPE` | 8 | core | 0 나눔 등 |
| `SIGKILL` | 9 | terminate | **catch 불가** |
| `SIGUSR1` | 10 | terminate | 사용자 정의 |
| `SIGSEGV` | 11 | core | 잘못된 메모리 접근 |
| `SIGUSR2` | 12 | terminate | 사용자 정의 |
| `SIGPIPE` | 13 | terminate | 끊긴 pipe 에 쓰기 |
| `SIGALRM` | 14 | terminate | `alarm()` |
| `SIGTERM` | 15 | terminate | 정상 종료 요청 |
| `SIGSTKFLT` | 16 | terminate | (사용 X) |
| `SIGCHLD` | 17 | ignore | 자식 상태 변경 |
| `SIGCONT` | 18 | continue | T → R |
| `SIGSTOP` | 19 | stop | **catch 불가** |
| `SIGTSTP` | 20 | stop | Ctrl+Z |
| `SIGTTIN/TTOU` | 21,22 | stop | bg 프로세스의 tty 접근 |
| `SIGURG` | 23 | ignore | TCP out-of-band |
| `SIGXCPU/XFSZ` | 24,25 | core | rlimit 초과 |
| `SIGIO` | 29 | terminate | I/O 가능 |
| `SIGSYS` | 31 | core | 잘못된 syscall (seccomp) |
| `SIGRTMIN..SIGRTMAX` | 32-64 | terminate | Real-time signals (queued) |

`kill -l` 로 OS 별 전체 리스트.

---

## 3. 시그널 보내기

```c
kill(pid, SIGTERM);
raise(SIGUSR1);              // 자신에게
pthread_kill(thread, SIG);   // 특정 스레드
killpg(pgrp, SIG);            // 프로세스 그룹
sigqueue(pid, SIG, value);    // siginfo 와 함께
```

```bash
kill -TERM 1234
kill -9 1234
kill -USR1 1234
pkill -USR1 nginx
killall -HUP rsyslogd
```

---

## 4. 시그널 처리

기본 동작:
- terminate
- terminate + core dump
- ignore
- stop
- continue

### 4.1 핸들러 설치 (옛 — 비추)

```c
#include <signal.h>
void handler(int sig) { ... }
signal(SIGINT, handler);
```

### 4.2 `sigaction()` (권장)

```c
struct sigaction sa = {0};
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;          // 차단된 syscall 재시작
sigaction(SIGINT, &sa, NULL);
```

| 플래그 | 의미 |
| --- | --- |
| `SA_RESTART` | EINTR 대신 syscall 자동 재시작 |
| `SA_SIGINFO` | siginfo_t 받는 handler |
| `SA_NOCLDWAIT` | SIGCHLD 무시 + 자동 zombie 회수 |
| `SA_NOCLDSTOP` | 자식 stop/cont 시 SIGCHLD 안 보냄 |
| `SA_ONSTACK` | 별도 stack 사용 (sigaltstack) |
| `SA_RESETHAND` | 1회만 사용 후 SIG_DFL |

### 4.3 SA_SIGINFO

```c
void handler(int sig, siginfo_t *info, void *uctx) {
    printf("sender pid=%d, value=%d\n", info->si_pid, info->si_value.sival_int);
}
sa.sa_flags = SA_SIGINFO;
sa.sa_sigaction = handler;
```

---

## 5. 시그널 mask

```c
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGINT);
pthread_sigmask(SIG_BLOCK, &set, NULL);    // 차단
pthread_sigmask(SIG_UNBLOCK, &set, NULL);
```

차단 동안 시그널 = **pending**. unblock 후 전달.

### 5.1 pending / blocked

```bash
cat /proc/$PID/status | grep -E 'SigQ|SigPnd|SigBlk|SigIgn|SigCgt'
```

---

## 6. 시그널 대기

### 6.1 pause / sigsuspend

```c
pause();                            // 어떤 시그널이든
sigsuspend(&mask);                  // atomic: mask 변경 + pause
```

### 6.2 sigwait / sigwaitinfo (스레드 친화)

```c
sigset_t set;
sigaddset(&set, SIGUSR1);
pthread_sigmask(SIG_BLOCK, &set, NULL);

int sig;
sigwait(&set, &sig);                // 동기 수신
```

→ 핸들러 X, 메인 흐름에서 수신. 멀티스레드에 추천 패턴.

### 6.3 signalfd (Linux)

```c
int sfd = signalfd(-1, &mask, 0);
// epoll 등 다른 FD 와 함께 select 가능
```

---

## 7. async-signal-safe 함수 ⚠️

시그널 핸들러는 **임의 순간** 에 실행. 일반 함수 호출 위험.

### 7.1 안전한 함수 (POSIX 명시)

```
write, read, _exit, signal, sigaction, kill, time, ...
일부 atomic / 단순
```

### 7.2 안전 X (절대 X)

```
malloc, free, printf, fprintf, fopen, ...
locale, locking 사용하는 모든 함수
```

→ 핸들러 안에선 **flag 만 설정** 후 메인이 처리:

```c
volatile sig_atomic_t got_signal = 0;
void handler(int s) { got_signal = 1; }
// 메인 루프에서 got_signal 검사
```

---

## 8. SIGTERM vs SIGKILL

| | SIGTERM | SIGKILL |
| --- | --- | --- |
| catch | 가능 | 불가 |
| cleanup | 가능 | 불가 |
| 권장 순서 | 1순위 | 최후 |

### 8.1 Graceful Shutdown 패턴

```c
signal(SIGTERM, on_term);
// SIGTERM 시 in-flight 요청 완료 + close FD + flush
```

Docker / Kubernetes 도 SIGTERM → grace period → SIGKILL.

---

## 9. SIGSEGV / SIGBUS — 메모리 오류

```
SIGSEGV — invalid 가상 주소 (NULL deref, stack overflow)
SIGBUS  — 정렬 오류, mmap 영역 밖
```

```bash
# core dump
ulimit -c unlimited
./crash
ls core.*
gdb ./crash core.1234
```

### 9.1 sigaltstack

stack overflow 시 SIGSEGV — 같은 stack 에서 handler 실행 불가.

```c
stack_t ss = { .ss_sp = malloc(SIGSTKSZ), .ss_size = SIGSTKSZ };
sigaltstack(&ss, NULL);
struct sigaction sa = { .sa_flags = SA_ONSTACK | SA_SIGINFO, .sa_sigaction = h };
sigaction(SIGSEGV, &sa, NULL);
```

---

## 10. SIGCHLD — 자식 상태 변경

```c
void on_chld(int sig) {
    int status;
    while (waitpid(-1, &status, WNOHANG) > 0) {
        // 모든 종료된 자식 회수
    }
}
sigaction(SIGCHLD, &(struct sigaction){.sa_handler=on_chld, .sa_flags=SA_RESTART|SA_NOCLDSTOP}, NULL);
```

→ 좀비 방지. 자세히 → [[orphan-zombie]]

---

## 11. Real-time Signal (32-64)

표준 시그널과 다름:
- **Queued** — 같은 시그널 여러 번 보내면 모두 전달
- **siginfo 의 value** 전달
- **순서 보장** (낮은 번호 우선)

```c
sigqueue(pid, SIGRTMIN+1, (union sigval){.sival_int = 42});
```

---

## 12. 시그널 vs 일반 IPC

| | 시그널 | IPC |
| --- | --- | --- |
| 데이터 | 비트 (+ 작은 value) | 임의 |
| 비동기 | ✅ | 보통 동기 (block) |
| 신뢰성 | 표준은 loss 가능 | 보장 |
| 용도 | 상태 알림 | 데이터 전달 |

복잡한 통신은 **socket / pipe / shm**.

---

## 13. 함정

### 함정 1 — printf in handler
async-signal-safe X. flag 만.

### 함정 2 — SIGKILL catch 시도
불가. SIGTERM 으로.

### 함정 3 — `signal()` 사용
오래된 API. semantics 가 OS 마다 다름. `sigaction()`.

### 함정 4 — `EINTR` 무시
시그널이 syscall 깨움 → EINTR. `SA_RESTART` 또는 loop.

### 함정 5 — SIGCHLD 핸들러에서 wait 1 번만
같은 순간에 N 자식 죽으면 N-1 좀비. **루프** + WNOHANG.

### 함정 6 — 멀티스레드 + 시그널
어느 스레드가 받을지 OS 결정. `pthread_sigmask` 로 명시 + sigwait.

### 함정 7 — Stack overflow → SIGSEGV
sigaltstack 없으면 handler 도 죽음.

### 함정 8 — fork 후 핸들러
복사됨. exec 후 SIG_DFL 로 reset.

---

## 14. 학습 자료

- **The Linux Programming Interface** Ch. 20-22
- `man 7 signal`, `man 7 signal-safety`
- POSIX 표준 — async-signal-safe 함수 목록

---

## 15. 관련

- [[fork-exec]] — fork 의 시그널 복사 / exec 의 reset
- [[orphan-zombie]] — SIGCHLD
- [[../syscall/syscall]] — kill / sigaction
- [[process]] — Process hub
