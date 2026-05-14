---
title: "Message ordering — sequence + 클럭 동기 X"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:06:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, ordering]
---

# Message ordering — sequence + 클럭 동기 X ★

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — server-assigned sequence

```sql
ALTER TABLE messages ADD COLUMN seq BIGINT NOT NULL;
CREATE UNIQUE INDEX ux_msg_room_seq ON messages (room_id, seq);
```

→ 각 room 별 `seq` increment (server-side assignment).
→ FE 가 ORDER BY seq.

---

## 2. 왜 client timestamp 안 씀

- 사용자 device 의 clock skew (5분+ 차이 가능).
- "A 의 폰" vs "B 의 폰" 시계 다름 → 메시지 순서 X.
- → **server-assigned sequence = single source of truth**.

---

## 3. 왜 created_at TIMESTAMPTZ 만으론 부족

- 같은 millisecond 안 2 메시지 → 순서 모호.
- partition 의 PK 가 (id, created_at) — id 는 ULID (시간 포함) but 같은 ms 가능.
- → seq 가 명시적.

---

## 4. seq 부여 전략

### Option A: room 별 sequence (PostgreSQL)

```sql
-- room 별 sequence 직접 fetch + lock
WITH next AS (
    UPDATE rooms SET last_seq = last_seq + 1
    WHERE id = $1
    RETURNING last_seq
)
INSERT INTO messages (id, room_id, seq, ...) VALUES (?, $1, (SELECT last_seq FROM next), ...);
```

→ row-level lock + atomic increment.

### Option B: Redis INCR

```java
long seq = redis.opsForValue().increment("room:seq:" + roomId);
```

→ 빠름 + Redis 의존.

### Option C: PostgreSQL 의 single sequence

```sql
CREATE SEQUENCE messages_global_seq;
```

→ room 별 X — 전역 ordering. chat 에는 부적합 (room 별 의미 X).

---

## 5. 본 vault 선택 — Option B (Redis INCR)

- 빠름 (서버 INSERT 직전 INCR).
- 본 vault 는 이미 Redis 사용 중 (presence / pub-sub).
- Redis fail 시 fallback: PostgreSQL atomic UPDATE.

---

## 6. FE 정렬

```javascript
// 메시지 list (REST GET)
messages.sort((a, b) => a.seq - b.seq);

// WebSocket receive 시 buffer
if (newMsg.seq === expectedSeq) {
    insertAtBottom(newMsg);
    expectedSeq++;
} else {
    // gap — out of order
    buffer.push(newMsg);
    if (gap too large) fetch missing via REST;
}
```

---

## 7. 함정

1. **client timestamp 사용** → clock skew 시 순서 X.
2. **room.last_seq 동시 INCR race** (Option A) → 같은 seq 2 메시지 → UNIQUE 충돌.
3. **Redis fail → PostgreSQL fallback 없음** → 메시지 영구 INSERT 실패.
4. **out-of-order broadcast** (멀티 디바이스) — Redis Pub/Sub 의 partition 이 보장 X.
   → Listener 가 seq 기준 정렬.
5. **FE buffer 무한** — gap 큰 메시지 영구 대기.
   → 5초 timeout + REST refetch.

---

## 8. 다른 컨텍스트

- 카톡: 자체 distributed sequence (Snowflake-like).
- WhatsApp: Erlang process per chat — 자연스러운 순서.
- Slack: timestamp + per-channel sequence.

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[../database/messages-table]]
- [[../implementation/message-send-impl]]
- [[../pitfalls/ordering-pitfalls]]
