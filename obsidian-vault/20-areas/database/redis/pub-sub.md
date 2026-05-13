---
title: "Redis Pub/Sub / Streams / Keyspace Notifications"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:50:00+09:00
tags:
  - database
  - redis
  - pubsub
  - streams
---

# Redis Pub/Sub / Streams / Keyspace Notifications

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Pub/Sub / Sharded / Streams / Notifications |

**[[redis|↑ Redis hub]]**

---

## 1. Pub/Sub vs Streams 한 줄

| 항목 | Pub/Sub | Streams |
| --- | --- | --- |
| 영속 | ❌ 휘발 | ✅ 영속 (메모리) |
| 과거 메시지 | ❌ | ✅ (consumer group) |
| 다중 consumer | ✅ (모두 받음) | ✅ (그룹별 partitioning) |
| Fan-out | 채널 broadcast | XREAD / consumer group |
| 손실 | 끊긴 동안 | XACK 안 하면 보관 |

→ **신뢰성 필요 = Streams** (또는 Kafka).
→ **이벤트 알림 (휘발) = Pub/Sub**.

---

## 2. Pub/Sub 기초

```
# subscriber
SUBSCRIBE news
SUBSCRIBE news sports

# publisher
PUBLISH news "hello"
PUBLISH sports "soccer score"

# pattern
PSUBSCRIBE news.*
PUBLISH news.kr "한국 뉴스"
```

| 명령 | 의미 |
| --- | --- |
| `SUBSCRIBE` | 채널 구독 |
| `UNSUBSCRIBE` | 해지 |
| `PSUBSCRIBE` | 패턴 구독 |
| `PUNSUBSCRIBE` | 패턴 해지 |
| `PUBLISH ch msg` | 발행 |
| `PUBSUB CHANNELS [pattern]` | 활성 채널 |
| `PUBSUB NUMSUB ch` | 구독자 수 |
| `PUBSUB NUMPAT` | 패턴 구독 수 |

---

## 3. Pub/Sub 의 한계

- **at-most-once** — 구독 안 한 동안 발행되면 안 받음
- **영속 X** — 디스크 X
- **Cluster** — 채널이 모든 노드에 broadcast (확장성 X)
- **fast 출구 필요** — 느린 구독자 → buffer 폭증

### 3.1 Sharded Pub/Sub (7.0+)

```
SSUBSCRIBE shardchannel
SPUBLISH shardchannel "msg"
```

Cluster 에서 채널이 hash slot 으로 매핑 → 한 노드 안에서만 전파.
→ Cluster 확장성 ↑.

---

## 4. Keyspace Notifications

```ini
notify-keyspace-events "Ex"        # 만료 이벤트만
notify-keyspace-events "AKE"       # 모두
```

| 문자 | 이벤트 |
| --- | --- |
| `K` | keyspace 채널 (`__keyspace@<db>__:key`) |
| `E` | keyevent 채널 (`__keyevent@<db>__:event`) |
| `g` | generic (DEL, EXPIRE, RENAME) |
| `$` | string |
| `l` | list |
| `s` | set |
| `h` | hash |
| `z` | sorted set |
| `x` | expired |
| `e` | evicted (maxmemory) |
| `m` | new keys |
| `A` | g$lshzxet (모두 제외 m) |

```bash
# 구독
PSUBSCRIBE '__keyevent@0__:expired'
# → "user:session:abc" expire 시 알림
```

사용처: TTL 끝난 키 후속 처리. **하지만 신뢰성 보장 X** — Pub/Sub 동안만.

⚠️ 모든 변경을 broadcast → 큰 워크로드에서 CPU ↑.

---

## 5. Streams — 신뢰성 메시징

### 5.1 추가

```
XADD stream * type "login" user "alice"
# * = 자동 ID (timestamp-seq)

# MAXLEN 으로 capped
XADD stream MAXLEN ~ 10000 * type "view" url "/home"
# ~ 는 "approximate" — 빠름
```

### 5.2 단순 소비 — XREAD

```
XREAD COUNT 10 BLOCK 0 STREAMS stream $
# $ = 새로운 메시지만
# 0 = 처음부터
# <id> = 그 이후부터
```

### 5.3 Consumer Group — Kafka-style

```
XGROUP CREATE stream mygroup $ MKSTREAM

# consumer-1
XREADGROUP GROUP mygroup consumer-1 COUNT 10 BLOCK 0 STREAMS stream >
# > = 그룹이 아직 못 받은 새 메시지

# 처리 후 ACK
XACK stream mygroup <id>

# 미처리 (consumer 죽음)
XPENDING stream mygroup
XAUTOCLAIM stream mygroup consumer-2 60000 0
```

### 5.4 Stream 의 장점

- 영속 (메모리 + AOF 시 디스크)
- 다수 consumer 분산 (group)
- 재처리 / 시간여행 가능 (id 로)
- Kafka 보다 가벼움 (한 Redis)

### 5.5 한계

- 디스크 X (Redis 메모리 한계)
- 거대 throughput (수십만 / 초) 은 Kafka

---

## 6. Pub/Sub vs Streams 결정

```
신뢰성 필요?
├── 예 → Streams (또는 Kafka)
└── 아니오 → Pub/Sub

지속 시간?
├── 휘발 OK → Pub/Sub
└── 과거 재생 / replay → Streams
```

---

## 7. Client Tracking — RESP3 (6.0+)

키 변경 시 클라이언트에 invalidation push.

```
CLIENT TRACKING ON
GET user:1
# 다른 클라이언트가 SET user:1 ...
# → 자동 invalidation notification
```

캐시 무효화에 강력. 클라이언트가 RESP3 지원 + 사용해야.

---

## 8. Pattern Matching

```
PSUBSCRIBE news.*           # news.kr, news.us, ...
PSUBSCRIBE __key*__:*       # 모든 keyspace
```

비효율 — 모든 채널 매칭. 가능하면 정확한 채널.

---

## 9. 실전 패턴

### 9.1 이벤트 알림 (휘발 OK)

```python
# publisher
r.publish("orders:new", json.dumps(order))

# subscriber
ps = r.pubsub()
ps.subscribe("orders:new")
for msg in ps.listen():
    if msg["type"] == "message":
        handle(msg["data"])
```

### 9.2 작업 큐 (Streams)

```python
# producer
r.xadd("jobs", {"task": "send_email", "to": "a@x.com"})

# worker
r.xgroup_create("jobs", "workers", id="$", mkstream=True)
while True:
    msgs = r.xreadgroup("workers", "worker-1",
                         {"jobs": ">"}, count=10, block=0)
    for stream, entries in msgs:
        for id, fields in entries:
            try:
                handle(fields)
                r.xack("jobs", "workers", id)
            except Exception:
                pass   # retry — XPENDING / XAUTOCLAIM
```

### 9.3 캐시 무효화 broadcast

```python
# DB 업데이트 시
r.set(f"cache:user:{uid}", json, ex=3600)
r.publish("cache:invalidate", f"user:{uid}")
```

### 9.4 만료 후처리

```python
ps = r.pubsub()
ps.psubscribe("__keyevent@0__:expired")
for msg in ps.listen():
    if msg["type"] == "pmessage":
        on_expire(msg["data"])
```

---

## 10. 함정

### 함정 1 — Pub/Sub 의 메시지 손실
구독 끊긴 동안 발행 = 사라짐. 신뢰성 필요 = Streams.

### 함정 2 — 느린 subscriber → buffer 폭증
`client-output-buffer-limit pubsub` 으로 강제 끊김. 빠른 처리 / 분기 worker.

### 함정 3 — Cluster + 일반 Pub/Sub
모든 노드 broadcast → 확장성 X. Sharded Pub/Sub.

### 함정 4 — Keyspace Notifications 의 비용
모든 변경 broadcast — 큰 워크로드 X. 필요한 이벤트만.

### 함정 5 — Streams 의 메모리 폭증
XADD MAXLEN 없이 무한 → 메모리 폭발. capped + trim.

### 함정 6 — XACK 누락
Pending list 폭증. 처리 후 반드시 XACK.

### 함정 7 — 만료 알림의 지연
Redis 의 만료는 lazy + active. 만료 직후가 아닐 수 있음.

---

## 11. 학습 자료

- **Redis Pub/Sub** — redis.io/docs/interact/pubsub
- **Redis Streams Introduction** — redis.io/docs/data-types/streams
- **Designing Data-Intensive Applications** Ch. 11 (Streams)

---

## 12. 관련

- [[data-types]] — Streams 기본
- [[commands]] — XADD/XREAD/XACK
- [[redis]] — Redis hub
