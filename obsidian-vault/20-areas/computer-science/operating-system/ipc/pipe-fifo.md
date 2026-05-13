---
title: "Pipe / FIFO — Byte Stream IPC"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:15:00+09:00
tags:
  - operating-system
  - ipc
  - pipe
---

# Pipe / FIFO

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | anonymous pipe + named FIFO |

**[[ipc|↑ IPC hub]]**

---

## 1. Anonymous Pipe

```c
int fd[2];
pipe(fd);              // fd[0] = read, fd[1] = write

if (fork() == 0) {
    close(fd[0]);
    write(fd[1], "hello", 5);
    close(fd[1]);
    _exit(0);
}
close(fd[1]);
char buf[16];
read(fd[0], buf, 16);
```

- **부모-자식** 또는 형제 (공유 부모) 사이만
- 단방향
- 양방향 필요시 pipe 2 개

### 1.1 pipe2 (Linux)

```c
pipe2(fd, O_NONBLOCK | O_CLOEXEC);
```

flag 와 동시에.

---

## 2. 셸의 pipe

```bash
ls | wc -l
```

```c
pipe(fd);
if (fork() == 0) {           // ls
    dup2(fd[1], 1);
    close(fd[0]); close(fd[1]);
    execlp("ls", "ls", NULL);
}
if (fork() == 0) {           // wc
    dup2(fd[0], 0);
    close(fd[0]); close(fd[1]);
    execlp("wc", "wc", "-l", NULL);
}
close(fd[0]); close(fd[1]);
wait(NULL); wait(NULL);
```

자세히 → [[../process/fork-exec#11-셸의-redirection-구현]]

---

## 3. FIFO (Named Pipe)

```bash
mkfifo /tmp/myfifo
# 또는 C: mkfifo("/tmp/myfifo", 0644)

# 한쪽에서
echo "hello" > /tmp/myfifo

# 다른쪽
cat /tmp/myfifo
```

- 무관한 프로세스 사이 — file system 의 이름
- 단방향 (양방향은 2 개)
- 파일이지만 데이터 없이 메모리 buffer 만

---

## 4. PIPE_BUF — atomic write

```
< PIPE_BUF (Linux 4096) byte = atomic — interleave X
≥ PIPE_BUF              = 다른 writer 와 interleave 가능
```

```c
#include <limits.h>
printf("%d\n", PIPE_BUF);    // 4096
```

→ 작은 메시지는 한 write 가 atomic. 큰 데이터는 framing 필요.

---

## 5. Buffer 크기

```bash
cat /proc/sys/fs/pipe-max-size           # 1048576 (1 MB 기본)
fcntl(fd, F_SETPIPE_SZ, 65536);           # 조정
fcntl(fd, F_GETPIPE_SZ);
```

기본 64 KB (Linux). buffer 가득 차면:
- writer 가 block (또는 EAGAIN)
- reader 가 비울 때까지

---

## 6. close 후 read

| 상태 | 동작 |
| --- | --- |
| writer 모두 close + buffer 비움 | read = 0 (EOF) |
| reader 모두 close, writer 가 write | SIGPIPE + EPIPE |

→ "쓰던 pipe 의 reader 죽음" = SIGPIPE → 보통 die. 방어:

```c
signal(SIGPIPE, SIG_IGN);
// 또는 send(..., MSG_NOSIGNAL)
```

---

## 7. Non-blocking pipe

```c
fcntl(fd, F_SETFL, O_NONBLOCK);
// read 비어있으면 EAGAIN, write 가득 차면 EAGAIN
```

이벤트 루프 (epoll) 와 결합.

---

## 8. splice — zero-copy pipe

```c
splice(in_fd, NULL, pipe_fd, NULL, sz, flags);
splice(pipe_fd, NULL, out_fd, NULL, sz, flags);
```

자세히 → [[../io/zero-copy#5-splice]]

→ pipe 가 kernel buffer 의 통로.

---

## 9. tee

```c
tee(pipe1, pipe2, sz, 0);
```

같은 데이터를 두 곳에 (둘 다 pipe).

`tee` 명령 (shell) 도 비슷한 의미.

---

## 10. FIFO vs Unix Socket

| | FIFO | Unix Socket |
| --- | --- | --- |
| 모델 | byte stream | stream / dgram |
| 양방향 | 1 개로는 X | ✅ |
| FD 전달 | X | ✅ (SCM_RIGHTS) |
| 권한 | file permission | + |
| 사용 | 단순 / 셸 / 1-way | client-server |

→ 복잡한 IPC = Unix socket 권장.

---

## 11. /dev/stdin / stdout / stderr 와의 통합

```c
// fd 0/1/2 도 그냥 FD — pipe 든 file 이든 동일
```

자식의 redirect 가 자연스러움.

---

## 12. /proc/$PID/fd/

```bash
ls -l /proc/$PID/fd/
# 3 -> pipe:[1234567]
# 4 -> /tmp/myfifo
```

pipe inode 번호 보임. 같은 inode 의 양쪽 = 같은 pipe.

---

## 13. 함정

### 13.1 SIGPIPE 무시 안 함
응용 죽음. `signal(SIGPIPE, SIG_IGN)` 또는 `MSG_NOSIGNAL`.

### 13.2 PIPE_BUF 초과 write
다른 writer 와 interleave. framing.

### 13.3 buffer 가득 + reader 없음
영원히 block. timeout / 비동기.

### 13.4 FIFO 의 open block
read open 은 writer 가 open 할 때까지 block (또는 O_NONBLOCK).

### 13.5 close 누락
EOF 안 보임 — reader 영원 대기.

### 13.6 FIFO 의 권한
file permission. 보안 / 권한 신중.

### 13.7 multiple readers
여러 reader 가 같은 FIFO read = race — 누가 받을지 X.

---

## 14. 학습 자료

- **The Linux Programming Interface** Ch. 44
- `man 7 pipe`, `man 7 fifo`
- **APUE** Ch. 15

---

## 15. 관련

- [[unix-socket]] — 더 강력한 IPC
- [[../process/fork-exec]] — 셸 pipe
- [[../io/zero-copy]] — splice
- [[ipc]] — IPC hub
