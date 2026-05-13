---
title: "IPC — Inter-Process Communication (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:10:00+09:00
tags:
  - operating-system
  - ipc
  - hub
---

# IPC (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 6 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

프로세스 간 데이터 / 신호 전달. 스레드는 메모리 공유 → IPC 불필요.

---

## 2. 종류 한눈에

| 메커니즘 | 단위 | 동기/비동기 | 호스트 | 노트 |
| --- | --- | --- | --- | --- |
| **Pipe** | byte stream | block / non-block | local | [[pipe-fifo]] |
| **FIFO (Named Pipe)** | byte stream | 같음 | local | [[pipe-fifo]] |
| **Unix Domain Socket** | stream / dgram | local | [[unix-socket]] |
| **TCP / UDP Socket** | stream / dgram | local + remote | — |
| **Shared Memory** | 임의 | manual sync | local | [[shared-memory]] |
| **Message Queue** | message | queued | local | [[message-queue]] |
| **Signal** | bit + small data | 비동기 알림 | local | [[signal]] |
| **Memory-mapped file** | 임의 | mmap | local | [[../memory/mmap]] |
| **D-Bus** | 메시지 | desktop | local | [[dbus]] |

---

## 3. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 부모 ↔ 자식 단방향 stream | pipe |
| 무관 프로세스 stream | FIFO / Unix socket |
| 같은 호스트 client-server | Unix socket |
| 다른 호스트 | TCP socket |
| 큰 데이터 (zero-copy) | shm + 동기화 |
| 메시지 큐 (분리 / 순서) | POSIX MQ / 외부 (RabbitMQ) |
| 상태 알림 | signal |
| 큰 파일 공유 | mmap 파일 |
| 데스크탑 / 시스템 서비스 | D-Bus |

---

## 4. 성능 비교 (개념)

```
shm + 자체 동기화  > Unix socket > Pipe > Message Queue > TCP socket (local) > 시그널
```

→ 큰 데이터 / 고성능 = shm.
→ 일반 / 안전 / 권한 = Unix socket.

---

## 5. 같은 호스트 socket vs Network socket

```c
// Unix domain socket
struct sockaddr_un addr = { .sun_family = AF_UNIX };
strcpy(addr.sun_path, "/tmp/myapp.sock");
```

- network stack 우회 → 더 빠름
- 권한 (file permission) + SCM_CREDENTIALS / SCM_RIGHTS
- localhost TCP 보다 항상 우위 (같은 호스트면)

---

## 6. FD 전달 (Unix socket only)

```c
sendmsg(sock, &msg, 0);   // SCM_RIGHTS 로 FD 전달
```

→ 한 프로세스가 다른 프로세스에 열린 FD 넘김. systemd socket activation, container runtime 등.

자세히 → [[unix-socket]]

---

## 7. 동기화

IPC 자체는 통신 — 동기화는 별도:
- Pipe / Socket — 자연스럽게 sync (read 가 block)
- shm — 별도 mutex / semaphore / atomic 필요
- MQ — queue 자체가 ordering

자세히 → [[../synchronization/synchronization]]

---

## 8. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[pipe-fifo]] | 익명 pipe + FIFO |
| [[shared-memory]] | shm / mmap / sysv shm |
| [[message-queue]] | POSIX MQ / sysv MQ |
| [[unix-socket]] | AF_UNIX / FD passing |
| [[signal]] | 시그널 (IPC 측면) |
| [[dbus]] | D-Bus / systemd 통신 |

---

## 9. 함정 (요약)

- pipe / FIFO의 PIPE_BUF (atomic write)
- shm 의 동기화 없음 → race
- POSIX MQ 의 사용 빈도 ↓
- signal 의 신뢰성 한계
- FD 전달 권한 검증
- Unix socket 파일의 cleanup

---

## 10. 학습 자료

- **The Linux Programming Interface** Ch. 43-57
- **APUE** (Advanced Programming in the UNIX Environment)
- **Beej's Guide to Unix IPC**

---

## 11. 관련

- [[../process/process]]
- [[../../network/network]]
- [[../memory/mmap]]
- [[../operating-system|↑ OS hub]]
