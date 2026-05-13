---
title: "I/O Management (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:10:00+09:00
tags:
  - operating-system
  - io
  - hub
---

# I/O Management (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 5 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

OS 가 응용 ↔ 디바이스 (디스크 / 네트워크 / 키보드 / GPU) 사이의 데이터 흐름을 추상화 + 관리.

---

## 2. I/O 모델

| 모델 | 동작 | 노트 |
| --- | --- | --- |
| **Blocking** | 결과까지 대기 | [[io-models]] |
| **Non-blocking** | 즉시 (EAGAIN) | [[io-models]] |
| **I/O Multiplexing** | select/poll/epoll | [[select-poll-epoll]] |
| **Signal-driven** | SIGIO | (드물게) |
| **Asynchronous (AIO)** | 완료 callback | [[io-uring]] |

---

## 3. 디바이스 종류

| 종류 | 의미 |
| --- | --- |
| **Block** | block 단위 (디스크) — random access |
| **Character** | byte 단위 (terminal, serial) |
| **Network** | socket (BSD) |
| **Pipe / FIFO** | byte stream |
| **Memory-mapped** | RAM 처럼 (mmap) |

`/dev/sda` block, `/dev/tty0` char, `/proc/...` virtual.

---

## 4. C10K → C10M

- **C10K** (1999, Dan Kegel) — 1 만 동시 연결 — select 의 한계 → epoll/kqueue 등장
- **C10M** (2013) — 1 천만 — kernel bypass / DPDK / userspace network

자세히 → [[../topics/c10k-c10m]] (작성 예정)

---

## 5. Zero-Copy

자세히 → [[zero-copy]]

```
일반: 디스크 → kernel → user buffer → kernel (socket) → NIC (4 copy)
zero-copy: 디스크 → kernel page cache → NIC (DMA, 1 copy)
```

`sendfile()`, `splice()`, `mmap+write`.

---

## 6. Page Cache + I/O

```
read():  디스크 → kernel page cache → user buffer (2 copy)
write(): user buffer → page cache (deferred to disk)
mmap():  page cache 직접 매핑 (0 copy)
O_DIRECT: page cache bypass
```

자세히 → [[../memory/mmap]], [[../filesystem/fsync-durability]]

---

## 7. 비동기 I/O

- POSIX AIO — 잘 안 씀
- Linux AIO (libaio) — direct I/O 만, 제한적
- **io_uring** (5.1+) — 표준 — 자세히 → [[io-uring]]
- Windows IOCP — 비슷한 idea

---

## 8. 디스크 스케줄러 (옛 / 일부)

```bash
cat /sys/block/sda/queue/scheduler
# none [mq-deadline] kyber bfq
```

| 스케줄러 | 의미 |
| --- | --- |
| `none` | 그대로 (NVMe 일반) |
| `mq-deadline` | deadline 기반 |
| `kyber` | latency 중심 |
| `bfq` | desktop 친화 (interactive 우선) |

SSD / NVMe = `none` 또는 mq-deadline 가 보통.

---

## 9. readahead

```bash
blockdev --getra /dev/sda            # 현재
blockdev --setra 256 /dev/sda         # 128 KB
```

순차 read 시 미리 fetch — 디스크 / fs 가 자동.

`posix_fadvise(POSIX_FADV_SEQUENTIAL)` 로 응용 힌트.

---

## 10. I/O 통계

```bash
iostat -x 1
# %util, await, svctm, r/s, w/s, rkB/s, wkB/s

pidstat -d 1
# 프로세스별 I/O

iotop
# top 처럼 I/O 별

cat /proc/diskstats
cat /proc/$PID/io
```

---

## 11. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[io-models]] | Blocking / Non-blocking / 비동기 |
| [[select-poll-epoll]] | I/O Multiplexing |
| [[io-uring]] | 모던 비동기 I/O |
| [[zero-copy]] | sendfile / splice / mmap |
| [[aio]] | POSIX AIO / libaio |

---

## 12. 응용 사례

| 시스템 | I/O 모델 |
| --- | --- |
| Nginx | epoll + non-blocking |
| Redis | epoll, single-thread loop |
| Node.js | libuv + epoll/kqueue |
| Tomcat (NIO) | epoll |
| ScyllaDB | io_uring + AIO |
| Tigerbeetle / FoundationDB | io_uring |
| io_uring 보급 후 | 점차 표준 |

---

## 13. 함정 (요약)

- Blocking I/O + 스레드 폭증 → 풀로 제한
- select 의 1024 FD 한계
- epoll 의 edge-trigger 핸들링
- O_DIRECT 정렬
- fsync 누락 → 손실
- TCP_NODELAY / Nagle 트레이드오프

---

## 14. 학습 자료

- **The Linux Programming Interface** Ch. 13, 59-63
- **OSTEP** Ch. 36-37
- **C10K** — Dan Kegel
- **io_uring** — Lord of the io_rings (LWN.net)

---

## 15. 관련

- [[../memory/mmap]] — page cache
- [[../filesystem/fsync-durability]]
- [[../../network/network]] — socket
- [[../operating-system|↑ OS hub]]
