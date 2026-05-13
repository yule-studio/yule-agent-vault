---
title: "Signal (IPC 관점)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:35:00+09:00
tags:
  - operating-system
  - ipc
  - signal
---

# Signal (IPC 관점)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | IPC 용도 / sigqueue / signalfd |

**[[ipc|↑ IPC hub]]**

---

## 1. IPC 로서의 시그널

전체 시그널 메커니즘은 → [[../process/signals]]
여기는 **IPC 측면** 만.

---

## 2. 시그널은 가장 단순한 IPC

- 1 bit 정보 (시그널 번호) — 일종의 비동기 알림
- 데이터 거의 X (sigqueue 의 value 만 32-bit)
- 신뢰성 부분적 (표준 signal 은 같은 종류 N 번 보내도 1 번)
- 매우 가벼움

→ **상태 변경 알림** 에 적합. 데이터 전달엔 부적합.

---

## 3. 보내기

```c
kill(pid, SIGUSR1);
killpg(pgrp, SIGTERM);
sigqueue(pid, SIGRTMIN+1, (union sigval){.sival_int = 42});
```

```bash
kill -USR1 1234
pkill -USR1 nginx
```

---

## 4. 받기 — 동기 (멀티스레드 친화)

```c
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGUSR1);
pthread_sigmask(SIG_BLOCK, &set, NULL);

int sig;
sigwait(&set, &sig);
// 처리
```

→ 핸들러 함정 (async-signal-safe) 없음.

---

## 5. signalfd — FD 로 받기

```c
sigset_t mask;
sigemptyset(&mask); sigaddset(&mask, SIGUSR1);
pthread_sigmask(SIG_BLOCK, &mask, NULL);

int sfd = signalfd(-1, &mask, 0);

struct signalfd_siginfo si;
read(sfd, &si, sizeof(si));
// si.ssi_signo, si.ssi_int (sigqueue 의 value)
```

→ `epoll` 등 이벤트 루프와 통합.

---

## 6. Real-time Signal — Queued

`SIGRTMIN ~ SIGRTMAX` (32-64):
- **여러 번 보내면 모두 전달** (표준 signal 은 1 번)
- siginfo 의 value 전달
- 우선순위 (낮은 번호 우선)

```c
sigqueue(pid, SIGRTMIN + 0, (union sigval){.sival_int = 100});
sigqueue(pid, SIGRTMIN + 1, (union sigval){.sival_int = 200});
```

→ 진짜 메시지 큐 비슷한 IPC.

---

## 7. Process Group / Session

```c
killpg(pgid, SIGTERM);             // 그룹 전체
kill(-pgid, SIGTERM);              // 같음
```

자식 process 트리 한 번에 종료 (셸의 `kill %1`).

---

## 8. SIGUSR1 / SIGUSR2 — 응용 정의

응용이 자유롭게 사용:
- daemon 의 reload 신호
- 디버그 정보 dump
- 자체 트리거

예 — nginx:
```
SIGHUP   → reload config
SIGUSR1  → reopen log files
SIGUSR2  → upgrade binary
SIGWINCH → graceful worker shutdown
```

---

## 9. SIGCHLD — 자식 알림

자세히 → [[../process/orphan-zombie#6-sigchld-로-자동-회수]]

부모가 자식의 상태 변화 (exit, stop, continue) 를 받는 IPC.

---

## 10. PR_SET_PDEATHSIG — 부모 죽으면

```c
prctl(PR_SET_PDEATHSIG, SIGTERM);
```

자식이 부모 죽으면 자동 시그널 수신. 백그라운드 워커 / cleanup.

---

## 11. signalfd vs sigwait vs handler

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| signalfd | epoll 통합, 명확 | 추가 FD |
| sigwait | 단순 | block 호출 필요 |
| handler | 자동 | async-signal-safe 함정 |

→ 멀티스레드 / 이벤트 루프 환경 = signalfd 권장.

---

## 12. 신뢰성 — 함정

```c
// kill -SIGUSR1 PID    1번 보냄
// kill -SIGUSR1 PID    2번 보냄

// 표준 signal — 응용은 1 번 받음 (pending bit)
```

해결:
- `SIGRTMIN+N` (queued)
- pipe / socket / shm (진짜 데이터)

→ 시그널은 "**무언가 일어났다**" 알림. 정확한 카운트 / 데이터 X.

---

## 13. 다른 IPC 와 결합 패턴

### 13.1 signal + pipe (self-pipe trick)

```c
int p[2]; pipe2(p, O_NONBLOCK);

void handler(int sig) {
    char c = sig;
    write(p[1], &c, 1);   // async-signal-safe
}

// main loop — epoll on p[0]
```

→ async-signal-safe 함수 부족 회피.

### 13.2 signalfd 가 표준 대안
self-pipe 의 모던 버전.

---

## 14. 함정

### 14.1 signal 안의 printf
async-signal-safe 함수만.

### 14.2 신뢰성 가정
표준 signal = lossy. RT signal 또는 다른 IPC.

### 14.3 SIGKILL 로 모든 종료
cleanup 없이. SIGTERM 우선.

### 14.4 fork 후 핸들러
복사됨. exec 후 default.

### 14.5 멀티스레드 + 시그널
어느 스레드 받을지 X. `pthread_sigmask` + sigwait.

### 14.6 SIGCHLD 핸들러 1번만 wait
N 자식 동시 종료 시 좀비.

### 14.7 SIGPIPE 무시 안 함
끊긴 pipe 쓰면 die.

---

## 15. 학습 자료

- **The Linux Programming Interface** Ch. 20-22
- **man 7 signal** + **man 7 signal-safety**
- [[../process/signals]] — 전체

---

## 16. 관련

- [[../process/signals]] — 모든 시그널
- [[pipe-fifo]] — self-pipe
- [[ipc]] — IPC hub
