---
title: "io_uring — Linux 모던 비동기 I/O"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:25:00+09:00
tags:
  - operating-system
  - io
  - io-uring
---

# io_uring

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | io_uring 개념 / API |

**[[io|↑ I/O hub]]**

---

## 1. 한 줄

Linux 5.1+ 의 **진정한 비동기 I/O**. 응용 ↔ 커널의 ring buffer 공유 → 시스템 콜 거의 없이 I/O.

Jens Axboe (Linux Block layer maintainer) 의 작품.

---

## 2. 왜

기존 Linux I/O 의 한계:
- `epoll` = readiness — 실제 I/O 는 syscall 마다
- `libaio` = direct I/O 만, 거의 비동기 X
- POSIX AIO = blocking 으로 fallback

io_uring 의 약속:
- 진짜 비동기 (block / network / fsync 모두)
- syscall 최소화 (batch submit + completion)
- 매우 빠름 (Spectre mitigation 부담 회피)

---

## 3. 구조

```
응용 (User)              Kernel
   ↓                       ↑
[Submission Queue (SQ)]  ←  consume + execute
                          submit
[Completion Queue (CQ)]  ←  push results
   ↑
   poll for completion
```

두 ring buffer 가 shared memory:
- SQ — 응용이 SQE (Submission Queue Entry) 작성
- CQ — 커널이 CQE (Completion Queue Entry) 작성

→ `io_uring_enter()` 1 회로 N 개 작업.

---

## 4. 핵심 API

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(256, &ring, 0);   // depth 256

// 작업 등록
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sz, offset);
io_uring_sqe_set_data(sqe, user_data);

io_uring_submit(&ring);

// 완료 대기
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
if (cqe->res < 0) { /* error */ }
io_uring_cqe_seen(&ring, cqe);
```

liburing 사용. 직접 syscall 도 가능.

---

## 5. 지원 op

- read / write / readv / writev
- accept / connect / send / recv
- fsync / fdatasync / sync_file_range
- openat / close / fadvise / madvise
- mkdir / rename / unlink
- splice / tee
- statx / lseek
- timeout / linked timeout

→ 거의 모든 I/O syscall.

---

## 6. polling 없는 완료 — IORING_SETUP_SQPOLL

```c
io_uring_queue_init_params(...);
// params.flags |= IORING_SETUP_SQPOLL;
```

- 커널이 별도 스레드로 SQ polling
- 응용이 `io_uring_enter` 도 거의 안 부름
- 극단의 저-latency

---

## 7. registered FD / buffer

```c
io_uring_register_files(&ring, fds, n);
io_uring_register_buffers(&ring, iovs, n);
```

- 매 SQE 마다 FD lookup / buffer copy 줄임
- 더 빠름

---

## 8. linked SQE

```c
sqe1->flags |= IOSQE_IO_LINK;
io_uring_prep_read(sqe1, fd, buf, sz, 0);
io_uring_prep_write(sqe2, fd2, buf, sz, 0);
// → sqe1 성공 후 sqe2 실행
```

순차 의존 작업.

---

## 9. epoll 과의 통합

io_uring 안에서 epoll 도 가능:

```c
io_uring_prep_poll_add(sqe, fd, POLLIN);
```

또는 io_uring fd 자체를 epoll 에 넣을 수도.

---

## 10. 성능

```
fio + io_uring: 1M+ IOPS / 코어 (NVMe)
fio + libaio:   ~ 500K IOPS
fio + sync:     ~ 100K IOPS
```

DB / 검색엔진 / 프록시에 큰 효과.

---

## 11. 사용 사례

| 시스템 | io_uring 사용 |
| --- | --- |
| **ScyllaDB** | seastar + io_uring |
| **TigerBeetle** | io_uring |
| **MySQL InnoDB** | 8.0+ 옵션 |
| **PostgreSQL** | 16+ partial |
| **Cassandra (4.1+)** | 옵션 |
| **fio** | 표준 backend |
| **iouring-net** (libuv) | 일부 |
| **Tokio** (Rust) | tokio-uring 별도 |

---

## 12. 커널 버전 별

| Linux | 기능 |
| --- | --- |
| 5.1 | 최초 |
| 5.4 | SQPOLL, fsync |
| 5.6 | timeout, async |
| 5.10 | LTS — 안정 |
| 5.15 | 더 많은 op |
| 6.0+ | 점점 빠르고 안전 |

⚠️ 옛 커널 (5.4 이하) = 보안 / 안정 우려 X. 5.15+ 권장.

---

## 13. 보안

io_uring 의 syscall 표면 → 일부 CVE 발견 (2022-2023).
Google Chrome 등은 sandbox 에서 io_uring 비활성.

```bash
sysctl kernel.io_uring_disabled = 1     # 비활성
```

production 에서 보안 모델 검토.

---

## 14. epoll vs io_uring

| | epoll | io_uring |
| --- | --- | --- |
| 모델 | Reactor | Proactor |
| readiness | yes | no (완료) |
| syscall | 매 op | batch |
| 디스크 I/O | sync | 비동기 |
| 학습 곡선 | 보통 | 가파름 |
| 보안 | 검증 | 새로움 |

→ 단순 epoll 로 충분한 경우 그대로. 극단 성능 / 디스크 비동기 = io_uring.

---

## 15. async runtime 의 활용

- Rust: `tokio-uring`, `monoio`, `glommio`
- Java: 표준 X — 일부 라이브러리
- Go: 일부 실험적 — 표준은 epoll
- Python: `aio-libs` 일부

대부분의 언어 표준 async 는 여전히 epoll. io_uring 은 커뮤니티 / 일부.

---

## 16. 함정

### 16.1 오래된 커널
사용 X. 5.15+ 권장.

### 16.2 보안 모델
sandbox 에서 io_uring → 추가 syscall 표면.

### 16.3 SQPOLL kernel thread
CPU 점유. 평가.

### 16.4 SQE 큐 가득
`io_uring_submit` 가 부분만 제출.

### 16.5 buffer / FD 의 lifecycle
완료 전 free 시 use-after-free.

### 16.6 라이브러리 부족
일부 언어는 ergonomic 한 wrapper X. liburing C API.

### 16.7 비동기 → 동기 fallback (옛 커널)
일부 op 가 fallback. 측정.

---

## 17. 학습 자료

- **Lord of the io_rings** — LWN.net (필독 시리즈)
- **liburing** — github.com/axboe/liburing
- **io_uring by example** — Shuveb Hussain
- **ScyllaDB / Seastar docs**

---

## 18. 관련

- [[io-models]]
- [[select-poll-epoll]]
- [[io]] — I/O hub
