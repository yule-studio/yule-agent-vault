---
title: "Message Queue — POSIX MQ / SysV MQ"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:25:00+09:00
tags:
  - operating-system
  - ipc
  - mq
---

# Message Queue

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | POSIX MQ + SysV MQ |

**[[ipc|↑ IPC hub]]**

---

## 1. 한 줄

프로세스 간 **메시지 단위** 통신. 큐 안에 줄 서서 FIFO + 우선순위 가능.

OS 의 mq vs 외부 broker (RabbitMQ / Kafka) — 같은 컨셉, 다른 규모.

---

## 2. POSIX MQ

```c
#include <mqueue.h>

// 생성
struct mq_attr attr = { .mq_maxmsg = 10, .mq_msgsize = 1024 };
mqd_t mq = mq_open("/myqueue", O_CREAT | O_RDWR, 0644, &attr);

// 송신 (우선순위 0 ~ 31)
mq_send(mq, "hello", 5, 0);
mq_timedsend(mq, "hello", 5, 0, &ts);

// 수신 (높은 우선순위부터)
char buf[1024];
unsigned prio;
ssize_t n = mq_receive(mq, buf, sizeof(buf), &prio);

// 비동기 알림
struct sigevent sev = { .sigev_notify = SIGEV_SIGNAL, .sigev_signo = SIGUSR1 };
mq_notify(mq, &sev);

// 정리
mq_close(mq);
mq_unlink("/myqueue");
```

특징:
- 이름 (`/myqueue`)
- 메시지 크기 / 큐 길이 attribute
- 우선순위 (0-31)
- block / non-block / timeout
- `select / poll / epoll` 가능 (Linux: mq_getfd via mqd_t)
- 비동기 통지 (signal / thread)

---

## 3. /dev/mqueue

```bash
mount | grep mqueue
# mqueue on /dev/mqueue type mqueue
ls /dev/mqueue
```

POSIX MQ 의 backing. 권한 / cleanup.

---

## 4. SysV MQ (옛)

```c
#include <sys/msg.h>

struct msgbuf { long mtype; char mtext[256]; };

key_t key = ftok("/path", 1);
int msqid = msgget(key, IPC_CREAT | 0644);

struct msgbuf buf = { .mtype = 1 };
strcpy(buf.mtext, "hello");
msgsnd(msqid, &buf, sizeof(buf.mtext), 0);

msgrcv(msqid, &buf, sizeof(buf.mtext), 0, 0);

msgctl(msqid, IPC_RMID, NULL);
```

- key + msqid
- `mtype` 으로 필터
- `ipcs -q` / `ipcrm -q`
- 거의 안 씀

---

## 5. 한계 / 제약

```bash
cat /proc/sys/fs/mqueue/queues_max         # 256 (per user)
cat /proc/sys/fs/mqueue/msg_max             # 10
cat /proc/sys/fs/mqueue/msgsize_max         # 8192

ipcs -l   # SysV
```

조정:
```bash
sysctl fs.mqueue.msg_max=1000
sysctl fs.mqueue.msgsize_max=65536
```

---

## 6. 우선순위

```c
mq_send(mq, "urgent", 6, 30);     // 높은 우선순위
mq_send(mq, "normal", 6, 5);

mq_receive(mq, buf, sz, &prio);    // urgent 먼저
```

→ 실시간 / 인터럽트 처리 시스템.

---

## 7. POSIX MQ vs Pipe / Socket

| | POSIX MQ | Pipe | Unix Socket |
| --- | --- | --- | --- |
| 단위 | 메시지 | byte stream | stream / dgram |
| 우선순위 | ✅ | ❌ | ❌ |
| 무관 프로세스 | ✅ (이름) | FIFO | ✅ |
| 비동기 통지 | ✅ | (poll) | (poll) |
| 보편성 | ★ (잘 안 씀) | ★★★★★ | ★★★★ |

→ MQ 의 우선순위 / 메시지 단위가 유용한 케이스만. 보통 socket 으로 충분.

---

## 8. 외부 Message Broker

OS MQ 의 한계:
- 같은 호스트만
- 영속 X
- 패턴 제한

해결 — 분산 message broker:
- **RabbitMQ** (AMQP) — 라우팅 풍부
- **Kafka** — 거대 throughput / 영속 / replay
- **NATS** — 가벼움
- **Redis Streams** — 작은 / 통합
- **AWS SQS / GCP Pub/Sub** — 매니지드

자세히 → [[../../network/rpc-messaging/mqtt-amqp-kafka]]

---

## 9. eventfd / pipe 로 가벼운 큐 흉내

```c
int efd = eventfd(0, EFD_SEMAPHORE);
uint64_t one = 1;
write(efd, &one, 8);            // signal
read(efd, &one, 8);              // wait
```

→ 카운터 기반의 가벼운 알림. epoll 친화.

```c
int p[2]; pipe2(p, O_NONBLOCK);
write(p[1], &msg, sizeof(msg));
read(p[0], &msg, sizeof(msg));
```

→ pipe 가 사실상 작은 메시지 큐 역할.

---

## 10. 응용 안의 메시지 큐 (라이브러리)

- C++ `boost::lockfree::queue`
- Java `ArrayBlockingQueue` / `LinkedBlockingQueue` / `Disruptor`
- Python `queue.Queue` / `multiprocessing.Queue`
- Go `chan`
- Rust `crossbeam-channel`

→ 같은 프로세스 안 — IPC 아님.

---

## 11. 함정

### 11.1 POSIX MQ 의 한계
일반 사용 X — pipe / socket 이 충분.

### 11.2 SysV MQ 의 cleanup
명시적 msgctl IPC_RMID — 없으면 reboot 까지 남음.

### 11.3 메시지 크기 한계
msgsize_max 초과 → EMSGSIZE.

### 11.4 큐 가득
mq_send block / EAGAIN. timeout.

### 11.5 권한 모델
SysV 의 uid/gid + perm. POSIX 가 더 단순.

### 11.6 우선순위 starvation
높은 우선순위만 도착 → 낮은 영원 대기. aging.

### 11.7 mq_notify 의 callback 함정
async-signal-safe 함수만 (signal handler 내).

---

## 12. 학습 자료

- **The Linux Programming Interface** Ch. 52
- `man 7 mq_overview`
- **POSIX MQ tutorial**

---

## 13. 관련

- [[unix-socket]] — 더 일반
- [[pipe-fifo]] — 단순
- [[../../network/rpc-messaging/mqtt-amqp-kafka]]
- [[ipc]] — IPC hub
