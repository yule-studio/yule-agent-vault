---
title: "Unix Domain Socket — AF_UNIX"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:30:00+09:00
tags:
  - operating-system
  - ipc
  - socket
---

# Unix Domain Socket

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | AF_UNIX / SOCK_STREAM / SOCK_DGRAM / SCM_RIGHTS |

**[[ipc|↑ IPC hub]]**

---

## 1. 한 줄

같은 호스트 내 프로세스 간 **socket API** 통신. TCP 와 같은 인터페이스지만 network stack 우회.

---

## 2. 종류

| Type | 의미 |
| --- | --- |
| `SOCK_STREAM` | TCP-like (순서 보장, byte stream) |
| `SOCK_DGRAM` | UDP-like (메시지, no order) |
| `SOCK_SEQPACKET` | 메시지 + 순서 보장 |

---

## 3. 주소

```c
#include <sys/un.h>

struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/myapp.sock");
```

### 3.1 Filesystem path

```
/tmp/myapp.sock
```

- 파일 시스템에 socket 노드 생성
- 권한 (chmod) 으로 접근 제어
- 응용 종료 시 unlink 필요

### 3.2 Abstract namespace (Linux)

```c
addr.sun_path[0] = '\0';
strcpy(addr.sun_path + 1, "myapp");
// length = strlen("myapp") + 1
```

- file system 노드 X
- 응용 종료 시 자동 정리
- container / sandbox 친화

`ss -x` 의 `@myapp` 처럼 표시.

---

## 4. Server

```c
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/app.sock");

unlink("/tmp/app.sock");                 // 이전 남은 것 정리
bind(sock, (struct sockaddr *)&addr, sizeof(addr));
listen(sock, 128);

while (1) {
    int cli = accept(sock, NULL, NULL);
    // 처리
    close(cli);
}
close(sock);
unlink("/tmp/app.sock");
```

---

## 5. Client

```c
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/app.sock");

connect(sock, (struct sockaddr *)&addr, sizeof(addr));
write(sock, "hello", 5);
close(sock);
```

→ TCP 코드와 거의 동일.

---

## 6. socketpair

```c
int fd[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, fd);

if (fork() == 0) {
    close(fd[0]);
    write(fd[1], "hi", 2);
    _exit(0);
}
close(fd[1]);
read(fd[0], buf, 2);
```

부모 ↔ 자식 양방향. pipe 의 양방향 버전.

---

## 7. FD 전달 — SCM_RIGHTS

다른 프로세스에 열린 FD 전달.

```c
struct msghdr msg = {0};
struct iovec iov = { .iov_base = "data", .iov_len = 4 };
msg.msg_iov = &iov;
msg.msg_iovlen = 1;

char cmsgbuf[CMSG_SPACE(sizeof(int))];
msg.msg_control = cmsgbuf;
msg.msg_controllen = sizeof(cmsgbuf);

struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_RIGHTS;
cmsg->cmsg_len = CMSG_LEN(sizeof(int));
memcpy(CMSG_DATA(cmsg), &send_fd, sizeof(int));

sendmsg(sock, &msg, 0);
```

→ 받은 프로세스의 FD table 에 새 FD 추가. 같은 file 의 reference.

### 7.1 사용처

- **systemd socket activation** — systemd 가 listen socket 만들고 FD 로 응용에 전달
- **Container runtime** — namespace / FD 전달
- **Chrome / Firefox** — 다중 process 의 GPU / network FD
- **Wayland** — display server ↔ client

---

## 8. SCM_CREDENTIALS — 피어 인증

```c
struct ucred {
    pid_t pid;
    uid_t uid;
    gid_t gid;
};

// 옵션 활성
int one = 1;
setsockopt(sock, SOL_SOCKET, SO_PASSCRED, &one, sizeof(one));

// recvmsg 시 ancillary data 로 ucred
```

→ peer 의 PID / UID 신뢰성 있게 확인. **TCP 보다 강한 인증** — kernel 이 검증.

대안: `getpeereid()` (BSD), `SO_PEERCRED` (Linux):

```c
struct ucred cred;
socklen_t len = sizeof(cred);
getsockopt(sock, SOL_SOCKET, SO_PEERCRED, &cred, &len);
```

---

## 9. TCP localhost vs Unix Socket

```
TCP 127.0.0.1:8080:
  network stack 통과 (TCP 핸드셰이크, congestion 등)
  성능 OK 이지만 overhead

Unix socket /tmp/x.sock:
  네트워크 우회
  10-30% 더 빠름
  권한 + 피어 인증
```

→ 같은 호스트면 Unix socket 우월.

```bash
ss -lx                   # listening Unix
ss -xp                    # 연결 + process
ss -lt                    # TCP
```

---

## 10. 응용

- **PostgreSQL** — `/var/run/postgresql/.s.PGSQL.5432`
- **MySQL** — `/var/run/mysqld/mysqld.sock`
- **Redis** — `unixsocket /tmp/redis.sock`
- **Docker** — `/var/run/docker.sock`
- **Kubernetes** — `containerd.sock` / `crio.sock`
- **systemd** — 거의 모든 service
- **D-Bus** — Unix socket 위
- **Wayland** — Unix socket
- **gRPC** — `unix:///path/to.sock`

---

## 11. 권한

```bash
ls -l /var/run/docker.sock
# srw-rw---- 1 root docker ...
```

- file permission 으로 접근
- `docker` 그룹에 사용자 추가
- `chmod 660` / `chown user:group`

⚠️ `srw-rw-rw-` 면 모든 사용자 — 위험.

---

## 12. EPIPE / SIGPIPE

TCP 와 동일 — 끊긴 socket 에 write → SIGPIPE.

```c
// 방어
signal(SIGPIPE, SIG_IGN);
// 또는
send(sock, ..., MSG_NOSIGNAL);
```

---

## 13. abstract socket 의 함정

- Linux 만
- container 내 격리되지만 — 같은 netns 의 모두 접근 가능 (filesystem 권한 X)
- network namespace 격리 필요

---

## 14. 함정

### 14.1 unlink 누락
다음 실행 시 bind 실패 (`EADDRINUSE`). 시작 시 unlink.

### 14.2 권한 너무 넓음
`0666` → 누구나. `0600` 또는 group.

### 14.3 path 길이 한계
`sockaddr_un.sun_path` 가 ~108 byte. 깊은 경로 X.

### 14.4 SIGPIPE 무시 안 함
응용 죽음.

### 14.5 SCM_RIGHTS 의 권한 검증 누락
받은 FD 의 적합성 응용이 확인.

### 14.6 abstract socket + container
의도와 다르게 노출 가능.

### 14.7 SOCK_DGRAM 의 message boundary
TCP 와 달리 partial X — buffer 너무 작으면 truncate.

### 14.8 socket file 백업
백업 도구가 socket 파일 통과 시도 — 보통 skip.

---

## 15. 학습 자료

- **The Linux Programming Interface** Ch. 57
- **Unix Network Programming** (Stevens) Ch. 15
- `man 7 unix`

---

## 16. 관련

- [[pipe-fifo]]
- [[../../network/network]]
- [[ipc]] — IPC hub
