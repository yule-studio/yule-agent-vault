---
title: "select / poll / epoll / kqueue — I/O Multiplexing"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:20:00+09:00
tags:
  - operating-system
  - io
  - epoll
---

# I/O Multiplexing — select / poll / epoll

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | select / poll / epoll / kqueue / IOCP |

**[[io|↑ I/O hub]]**

---

## 1. 한 줄

한 스레드가 **여러 FD 의 상태** 를 한 syscall 로 감시. C10K 의 해답.

---

## 2. select (POSIX, 1983)

```c
fd_set rfds; FD_ZERO(&rfds);
FD_SET(fd1, &rfds);
FD_SET(fd2, &rfds);

struct timeval tv = {5, 0};   // 5초 timeout
int n = select(max_fd + 1, &rfds, NULL, NULL, &tv);

if (FD_ISSET(fd1, &rfds)) read(fd1, ...);
```

### 2.1 한계
- FD_SETSIZE = **1024** (보통). 컴파일 시 고정.
- 매 호출마다 전체 fd_set 복사 (커널 ↔ 유저)
- O(N) 스캔
- 호출 시마다 fd_set 재구성

→ C10K 이상 불가.

---

## 3. poll (POSIX)

```c
struct pollfd fds[N];
fds[0].fd = sock; fds[0].events = POLLIN;
fds[1].fd = stdin; fds[1].events = POLLIN;

int n = poll(fds, N, 5000);

if (fds[0].revents & POLLIN) read(sock, ...);
```

### 3.1 차이
- FD 수 제한 X (배열 크기)
- 같은 O(N) 비용, 같은 복사

→ select 보다 약간 나음. 대안 epoll 이 진짜 해답.

---

## 4. epoll (Linux 2.5.45+, 2002)

```c
int epfd = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = sock };
epoll_ctl(epfd, EPOLL_CTL_ADD, sock, &ev);

struct epoll_event events[256];
int n = epoll_wait(epfd, events, 256, 5000);
for (int i = 0; i < n; i++) {
    if (events[i].events & EPOLLIN)
        read(events[i].data.fd, ...);
}
```

### 4.1 특징
- **준비된 FD 만** 반환 (O(ready))
- FD 등록 한 번 (변경 시만)
- 커널이 RB tree + ready list 관리
- 수십만 FD OK

### 4.2 Level Trigger (LT) vs Edge Trigger (ET)

| | LT (기본) | ET |
| --- | --- | --- |
| 의미 | 상태가 ready 인 동안 계속 알림 | 상태가 변할 때만 |
| 활용 | 간편 | EAGAIN 까지 모두 처리 필요 |
| 효율 | 약간 ↓ | ↑ |

```c
ev.events = EPOLLIN | EPOLLET;
```

### 4.3 ET 의 패턴

```c
while ((n = read(fd, buf, sz)) > 0) {
    // 처리
}
if (n < 0 && errno == EAGAIN) {
    // 다시 epoll_wait
}
```

→ ET 면 한 번에 모두 read. 안 그러면 다음 알림 X.

### 4.4 EPOLLONESHOT
한 번만 알림. 처리 후 다시 EPOLL_CTL_MOD.

### 4.5 EPOLLEXCLUSIVE
같은 FD 를 여러 epoll 이 감시 시 wake 1 명만 (thundering herd 방지).

---

## 5. kqueue (BSD / macOS, 2000)

```c
int kq = kqueue();
struct kevent ev;
EV_SET(&ev, sock, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &ev, 1, NULL, 0, NULL);

struct kevent events[256];
int n = kevent(kq, NULL, 0, events, 256, NULL);
```

- epoll 비슷 + 더 일반 (file, signal, timer, vnode 등 모두)
- macOS / FreeBSD 표준

---

## 6. Windows IOCP

I/O Completion Port — Proactor 모델.

```c
HANDLE iocp = CreateIoCompletionPort(...);
ReadFile(handle, buf, sz, NULL, &overlapped);

DWORD bytes;
ULONG_PTR key;
LPOVERLAPPED ov;
GetQueuedCompletionStatus(iocp, &bytes, &key, &ov, INFINITE);
```

- 완료 알림 (Proactor)
- ASIO / .NET / Windows 서버

---

## 7. 비교

| 항목 | select | poll | epoll | kqueue | IOCP |
| --- | --- | --- | --- | --- | --- |
| OS | POSIX | POSIX | Linux | BSD/macOS | Windows |
| 모델 | Reactor | Reactor | Reactor | Reactor | Proactor |
| FD 제한 | 1024 | 무 | 무 | 무 | 무 |
| 비용 | O(N) | O(N) | O(ready) | O(ready) | O(complete) |
| 등록 | 매 호출 | 매 호출 | 1 번 | 1 번 | 매 op |

---

## 8. 라이브러리 추상

- **libev** — 가벼움
- **libevent** — 기능 풍부
- **libuv** — Node.js + 크로스 플랫폼
- **Boost.ASIO** — C++ + IOCP/epoll/kqueue 통합
- **Tokio / mio** — Rust 의 epoll/kqueue/IOCP

대부분 응용은 직접 epoll 호출 X — 라이브러리.

---

## 9. Edge Trigger 의 함정

### 9.1 EAGAIN 처리

```c
// ❌ ET 인데 1번만 read
read(fd, buf, 4096);
// → 나머지 데이터 있어도 epoll_wait 가 안 알림

// ✅ EAGAIN 까지
while ((n = read(fd, buf, 4096)) > 0) {
    process(buf, n);
}
if (n < 0 && errno != EAGAIN) abort();
```

### 9.2 partial write
- write 시도 → 일부만 — EAGAIN
- 남은 buffer 보관 + EPOLLOUT 등록 → 가능해지면 다시

---

## 10. Thundering Herd

```
한 listen socket 을 여러 worker 가 epoll → 새 연결 → 모두 깨어남
```

- accept 가 한 명만 성공, 나머지는 EAGAIN
- Linux 4.5+ `EPOLLEXCLUSIVE` 로 해결
- Nginx 는 `SO_REUSEPORT` (4.5+) 로 worker 별 listen socket → wake 1

---

## 11. SO_REUSEPORT

```c
int opt = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
bind(sock, ...);
```

여러 worker 가 같은 port 에 bind → 커널이 load balance.
→ NGINX / HAProxy / Envoy 표준.

---

## 12. eventfd / timerfd / signalfd

```c
int efd = eventfd(0, EFD_NONBLOCK);
int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
int sfd = signalfd(-1, &mask, 0);
```

- 이벤트 / 타이머 / 시그널 도 FD → epoll 로 통합 감시
- 이벤트 루프의 단일 진입점

---

## 13. Single-thread event loop 패턴

```c
while (running) {
    int n = epoll_wait(epfd, events, MAX, timeout);
    for (int i = 0; i < n; i++) {
        if (events[i].events & EPOLLIN)  handle_read(events[i].data.fd);
        if (events[i].events & EPOLLOUT) handle_write(events[i].data.fd);
    }
    process_timers();
}
```

- 단일 스레드 (또는 코어 수 만큼)
- 모든 작업 비동기 / 짧게
- CPU 무거운 작업 → 별도 thread pool

→ Redis 의 단순 모델.

---

## 14. 함정

### 14.1 select 사용
2026 년에 거의 이유 없음. epoll / kqueue / libuv.

### 14.2 ET + partial read
EAGAIN 까지. 안 그러면 starvation.

### 14.3 LT + busy
ready 인데 처리 안 함 → 매 wait 가 즉시 return.

### 14.4 epoll 의 deleted FD
close 한 FD 가 epoll set 에 남음 → 다음 dup 한 같은 번호의 FD 가 잘못 알림. close 전 EPOLL_CTL_DEL.

### 14.5 단일 스레드 안에서 blocking call
이벤트 루프 멈춤. spawn 별도 스레드.

### 14.6 thundering herd
EPOLLEXCLUSIVE / SO_REUSEPORT.

### 14.7 EPOLLET + EPOLLONESHOT 동시
rearm 누락 시 영원히 안 알림.

---

## 15. 학습 자료

- **The C10K problem** — Dan Kegel
- **The Linux Programming Interface** Ch. 63
- **Unix Network Programming** (Stevens) Ch. 6
- **brendangregg.com — epoll / Network IO**
- **libuv internals**

---

## 16. 관련

- [[io-models]]
- [[io-uring]]
- [[../topics]] — C10K
- [[io]] — I/O hub
