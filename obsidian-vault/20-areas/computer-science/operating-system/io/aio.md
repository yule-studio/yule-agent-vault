---
title: "AIO — POSIX AIO / Linux libaio"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:35:00+09:00
tags:
  - operating-system
  - io
  - aio
---

# AIO — POSIX AIO / Linux libaio

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | AIO 비교 + io_uring 으로 이전 |

**[[io|↑ I/O hub]]**

---

## 1. 한 줄

비동기 I/O 의 옛 표준. **Linux 에서는 거의 사용 X** — io_uring 으로 대체.

---

## 2. POSIX AIO

```c
#include <aio.h>

struct aiocb cb = {0};
cb.aio_fildes = fd;
cb.aio_buf = buf;
cb.aio_nbytes = sz;
cb.aio_offset = 0;

aio_read(&cb);

while (aio_error(&cb) == EINPROGRESS) ;
ssize_t n = aio_return(&cb);
```

### 2.1 Linux glibc 구현
- thread pool 로 emulate (실제로 동기)
- 진짜 비동기 X
- 실용성 ↓

→ Linux 에선 사실상 사용 X.

### 2.2 BSD / Solaris
일부에선 진짜 비동기 구현.

---

## 3. Linux libaio (kernel aio)

```c
#include <libaio.h>

io_context_t ctx;
io_setup(128, &ctx);

struct iocb iocb;
io_prep_pread(&iocb, fd, buf, sz, offset);
struct iocb *iocbs[] = { &iocb };
io_submit(ctx, 1, iocbs);

struct io_event events[1];
io_getevents(ctx, 1, 1, events, NULL);

io_destroy(ctx);
```

### 3.1 제약
- `O_DIRECT` 만 진짜 비동기 — buffered I/O 는 sync fallback
- 메모리 / 정렬 strict
- 일부 syscall 비동기 X (fsync 등)

### 3.2 사용처
- MySQL InnoDB (옛)
- 일부 DB

→ io_uring 등장 후 점차 마이그.

---

## 4. io_uring 의 우월성

자세히 → [[io-uring]]

| | libaio | io_uring |
| --- | --- | --- |
| 진짜 비동기 | O_DIRECT 만 | 거의 모두 |
| op 종류 | 적음 | 매우 다양 |
| Syscall | 많음 | 적음 (batch) |
| 보안 | 검증 | 새로움 (CVE) |

→ 신규는 io_uring. 옛 시스템 유지보수는 libaio.

---

## 5. Windows IOCP

```c
HANDLE iocp = CreateIoCompletionPort(...);
ReadFile(handle, buf, sz, NULL, &overlapped);

DWORD bytes;
ULONG_PTR key;
LPOVERLAPPED ov;
GetQueuedCompletionStatus(iocp, &bytes, &key, &ov, INFINITE);
```

- **Proactor** (완료 알림)
- 진짜 비동기
- ASP.NET / Windows 서버 표준
- Boost.ASIO / .NET Async 의 backend

→ io_uring 의 Windows 버전 비슷.

---

## 6. Thread Pool 로 비동기 흉내

```c
// 가짜 비동기 — pool 의 worker 가 동기 read
void async_read(int fd, void *buf, size_t sz, callback cb) {
    pool_submit(pool, [=]{
        ssize_t n = read(fd, buf, sz);
        cb(n);
    });
}
```

- POSIX AIO 의 glibc 구현 = 이거
- libuv 일부 (디스크)
- 단순 / 광범위 호환

문제 — context switch 비용, latency.

---

## 7. 응용 framework

| Framework | 비동기 모델 |
| --- | --- |
| Node.js / libuv | epoll + thread pool (디스크) |
| Boost.ASIO | IOCP / epoll / kqueue 추상화 |
| Tokio (Rust) | mio (epoll/kqueue) |
| tokio-uring | io_uring |
| Java NIO | epoll/kqueue/IOCP |
| Java NIO.2 (AsyncChannel) | epoll + thread pool |
| Netty | epoll/kqueue + native |

io_uring 으로 점차 마이그.

---

## 8. 함정

### 8.1 POSIX AIO + Linux
glibc 의 thread pool. 진짜 비동기 X.

### 8.2 libaio + buffered I/O
sync fallback. O_DIRECT 필수.

### 8.3 O_DIRECT 정렬 누락
EINVAL.

### 8.4 io_setup limit
`fs.aio-max-nr` sysctl. 부족 시 EAGAIN.

### 8.5 신규 코드 libaio
io_uring 권장.

### 8.6 IOCP + thread pool 의 차이
IOCP 의 worker 가 IOCP 가 부르는 thread 인지 응용이 만든 thread pool 인지.

---

## 9. 학습 자료

- **The Linux Programming Interface** Ch. 41
- **libaio man page**
- **Lord of the io_rings** — LWN
- **POSIX AIO** docs

---

## 10. 관련

- [[io-uring]]
- [[io-models]]
- [[io]] — I/O hub
