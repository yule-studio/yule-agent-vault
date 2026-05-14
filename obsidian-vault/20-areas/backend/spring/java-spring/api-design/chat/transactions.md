---
title: "chat transactions — race + 순서 + idempotency"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:52:00+09:00
tags: [backend, java-spring, api-design, chat, transactions]
---

# chat transactions — race + 순서 + idempotency

**[[chat|↑ hub]]**

---

## 1. 격리

| 영역 | 격리 |
| --- | --- |
| 메시지 INSERT | READ_COMMITTED |
| 읽음 표시 INSERT | READ_COMMITTED + UNIQUE |
| 멤버 join/leave | READ_COMMITTED + version |
| presence | Redis (DB X) |

---

## 2. Race 시나리오

| 시나리오 | 방어 |
| --- | --- |
| 같은 message 의 동시 READ INSERT (멀티 디바이스) | UNIQUE (message_id, user_id) — ON CONFLICT DO NOTHING |
| 같은 room 의 같은 user 가 동시 send 2개 | sequence number 증가 + send time |
| 멤버 동시 join (max 초과) | room.version optimistic |
| 메시지 send 도중 user blocked | sender 의 차단 list query → throw |
| Redis publish 도중 server crash | 메시지 DB 는 있고 broadcast X — 다른 user 가 GET / 페이징 시 받음 |

---

## 3. 트랜잭션 경계

### 3.1 메시지 send

```
@Transactional
1. room.canSend(user) — 멤버 / 차단 검증
2. message INSERT (seq 증가)
3. publishEvent (MessageSent) — AFTER_COMMIT
```

AFTER_COMMIT:
- Redis publish (cross-node fan-out)
- offline member 의 notification outbox INSERT
- WebSocket broadcast

→ 트랜잭션 밖 외부 호출 (Redis / notification).

### 3.2 읽음 표시

```
@Transactional
1. message_read INSERT (ON CONFLICT DO NOTHING)
2. publishEvent (MessageRead) — AFTER_COMMIT
```

AFTER_COMMIT:
- Redis publish (다른 device + 다른 user 의 "읽음 카운트" 업데이트)

### 3.3 WebSocket session register

```
On WebSocket CONNECT:
  Redis SADD "user:{userId}:sessions" {sessionId}
  Redis EXPIRE 1h

On DISCONNECT:
  Redis SREM
```

→ DB X (Redis 만).

---

## 4. 순서 보장 (메시지)

### 4.1 같은 room → server-side seq

```sql
CREATE SEQUENCE messages_seq_per_room;   -- room 별 별도 sequence
-- 또는: messages 테이블에 seq BIGINT + UNIQUE (room_id, seq)
```

→ DB 의 unique sequence 가 source of truth.
→ client 는 seq 기준 정렬.

### 4.2 멀티 디바이스 의 같은 사용자 동시 send

- A 가 폰 + 태블릿 에서 동시 "안녕" send → 둘 다 INSERT (다른 message_id, 다른 seq).
- → 사용자가 2번 보낸 것 (의도) — 합치지 X.

---

## 5. Multi-device idempotency (옵션)

- FE 가 `clientMessageId` (UUID) 부여 → server 가 UNIQUE check.
- 같은 device 에서 더블 클릭 시 → 두 번째 INSERT 실패 (idempotent).

```sql
ALTER TABLE messages ADD COLUMN client_message_id VARCHAR(50);
CREATE UNIQUE INDEX ux_messages_client ON messages (sender_id, client_message_id)
    WHERE client_message_id IS NOT NULL;
```

---

## 6. 함정

- 메시지 send 트랜잭션 안 Redis publish — 외부 호출.
- 메시지 broadcast 후 DB INSERT — broadcast 받았는데 DB 없음 (사용자 reload 시 사라짐).
- READ UNIQUE 없음 — 멀티 디바이스 동시 READ → 중복 row.
- room.member 의 version 없음 — 동시 join max 초과.

---

## 7. 관련

- [[chat|↑ hub]]
- [[design-decisions/message-ordering]]
- [[design-decisions/multi-device-sync]]
- [[implementation/message-send-impl]]
- [[pitfalls/ordering-pitfalls]]
