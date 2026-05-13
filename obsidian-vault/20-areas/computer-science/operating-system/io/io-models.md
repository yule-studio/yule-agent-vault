---
title: "I/O 모델 — Blocking / Non-blocking / Async"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:15:00+09:00
tags:
  - operating-system
  - io
  - blocking
---

# I/O 모델

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 5 모델 비교 |

**[[io|↑ I/O hub]]**

---

## 1. 5 가지 모델 (Stevens)

| 모델 | wait | 알림 |
| --- | --- | --- |
| **Blocking** | 응용 멈춤 | return |
| **Non-blocking (Polling)** | 즉시 return EAGAIN | 응용이 다시 시도 |
| **I/O Multiplexing** | select/poll/epoll | ready event |
| **Signal-driven** | continue | SIGIO |
| **Asynchronous I/O** | continue | 완료 시 callback / event |

---

## 2. Blocking

```c
ssize_t n = read(fd, buf, 4096);   // 데이터 올 때까지 대기
// 그 동안 스레드 sleep
```

- 단순
- 1 연결 = 1 스레드 → 수천 동시 = 수천 스레드 = stack × N + cs 폭증
- **C10K 문제** 의 원인

---

## 3. Non-blocking (Polling)

```c
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

ssize_t n = read(fd, buf, 4096);
if (n < 0 && errno == EAGAIN) {
    // 데이터 없음 — 다시 try
}
```

- busy poll = CPU 폭증
- 단독은 거의 안 씀 — multiplexing 과 결합

---

## 4. I/O Multiplexing — select / poll / epoll

```c
fd_set rfds; FD_ZERO(&rfds);
FD_SET(fd1, &rfds);
FD_SET(fd2, &rfds);
select(maxfd+1, &rfds, NULL, NULL, NULL);   // 한 스레드가 N FD 감시

if (FD_ISSET(fd1, &rfds)) read(fd1, ...);
```

자세히 → [[select-poll-epoll]]

- 1 스레드 N 연결 ← C10K 의 해답
- Nginx / Redis / Node.js / Go 의 기반

---

## 5. Signal-driven (SIGIO)

```c
fcntl(fd, F_SETOWN, getpid());
fcntl(fd, F_SETFL, O_ASYNC);
// 데이터 준비 → SIGIO 발송 → 핸들러 호출
```

- 핸들러의 제약 (async-signal-safe)
- 거의 안 씀

---

## 6. Asynchronous I/O

응용이 요청 → OS 가 진짜로 비동기 처리 → 완료 시 알림.

### 6.1 POSIX AIO

```c
struct aiocb cb = {0};
cb.aio_fildes = fd;
cb.aio_buf = buf;
cb.aio_nbytes = sz;
aio_read(&cb);
// 비동기
while (aio_error(&cb) == EINPROGRESS) ;
aio_return(&cb);
```

→ 옛 — Linux 구현이 약함, 거의 사용 X.

### 6.2 Linux libaio

```c
io_setup(...);
io_submit(...);
io_getevents(...);
```

`O_DIRECT` 만 진짜 비동기. buffered I/O 는 동기 fallback. DB 가 일부 사용.

### 6.3 io_uring (5.1+)

표준 — 자세히 → [[io-uring]]

### 6.4 Windows IOCP

I/O Completion Port. Windows 의 표준 비동기.

---

## 7. 동기 vs 비동기

| | 동기 | 비동기 |
| --- | --- | --- |
| 호출 후 | wait | return |
| 결과 | return 으로 | callback / event |
| 코드 | 직관적 | 복잡 (callback hell) |

→ async/await 가 callback hell 완화.

---

## 8. Blocking vs Non-blocking

| | Blocking | Non-blocking |
| --- | --- | --- |
| FD 모드 | default | `O_NONBLOCK` |
| read 못 함 시 | 대기 | EAGAIN |

직교 개념:
- Blocking + 동기 = `read()` 단독
- Non-blocking + 동기 = `read()` + poll
- Non-blocking + 비동기 = io_uring 등

---

## 9. 1 스레드 N 연결 vs N 스레드 N 연결

### 9.1 Thread per connection (전통)
- 단순
- 1 GB stack × 10000 = 10 TB (불가)
- context switch 폭증

### 9.2 Event loop (Reactor)
- 1 (또는 적은 수) 스레드 + epoll
- 빠르지만 callback 모델
- Nginx / Redis / Node

### 9.3 Async/await
- Event loop + 언어 지원 — 직관적 코드
- Python asyncio / Rust tokio / C# async / JS

### 9.4 N:M (goroutine / virtual thread)
- 코드는 동기적 — 런타임이 비동기로 분해
- Go / Java Loom

자세히 → [[../threads/thread-models]]

---

## 10. Reactor vs Proactor

| | Reactor | Proactor |
| --- | --- | --- |
| 이벤트 | "ready to read" | "completed" |
| 응용 책임 | 직접 read | 결과 받기 |
| 예 | epoll | IOCP / io_uring |

io_uring 이 Linux 에 Proactor 를 들여옴.

---

## 11. 실전 권장

| 시나리오 | 모델 |
| --- | --- |
| 적은 동시 (수십) | thread + blocking |
| 많은 동시 (수만) | epoll / async |
| latency 극단 | io_uring + busy poll |
| 디스크 I/O 많음 | AIO / io_uring |
| 단순 CLI | blocking |

---

## 12. 함정

### 12.1 Non-blocking + 무한 retry
busy poll → CPU 100%. multiplexing 필수.

### 12.2 Blocking 스레드 폭증
풀로 제한.

### 12.3 epoll edge-trigger 의 partial read
EAGAIN 까지 모두 read. 자세히 → [[select-poll-epoll]]

### 12.4 async 안의 sync call
이벤트 루프 막힘. spawn_blocking / 별도 스레드.

### 12.5 AIO 가 blocking 으로 fallback
buffered I/O 면 libaio 가 동기. O_DIRECT.

### 12.6 partial read / partial write
read/write 가 요청한 만큼 안 줄 수 있음 — 루프.

---

## 13. 학습 자료

- **Unix Network Programming** (Stevens) Ch. 6
- **The C10K problem** — Dan Kegel
- **Java NIO** / **node.js** docs

---

## 14. 관련

- [[select-poll-epoll]]
- [[io-uring]]
- [[../threads/thread-models]]
- [[io]] — I/O hub
