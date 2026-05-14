---
title: "message_reads 테이블 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:38:00+09:00
tags: [backend, java-spring, api-design, chat, database, read]
---

# message_reads 테이블 ★

**[[database|↑ hub]]**

> 본 vault — per-user per-room `last_read_seq` (compact) + 옵션 per-message READ row (그룹 "본 사람 list").

---

## 1. Schema (compact — 본 vault 기본)

```sql
CREATE TABLE room_member_read_state (
    room_id        CHAR(26) NOT NULL,
    user_id        CHAR(26) NOT NULL,
    last_read_seq  BIGINT NOT NULL DEFAULT 0,
    last_read_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (room_id, user_id)
);

CREATE INDEX ix_rmrs_user ON room_member_read_state (user_id);
```

→ room_members.last_read_seq 컬럼으로 통합 가능 (본 vault 는 통합).

---

## 2. Schema (옵션 — per-message)

```sql
-- 그룹 의 "본 사람 list" 표시용 (옵션)
CREATE TABLE message_reads (
    message_id   CHAR(26) NOT NULL,
    user_id      CHAR(26) NOT NULL,
    read_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (message_id, user_id)
);

CREATE INDEX ix_mr_message ON message_reads (message_id);
CREATE INDEX ix_mr_user ON message_reads (user_id, read_at DESC);
```

→ UNIQUE PK = 멀티 디바이스 동시 read 시 ON CONFLICT DO NOTHING.

---

## 3. "안 읽은 수" 계산

```sql
-- 내 메시지의 안 읽은 수 (1:1)
SELECT COUNT(*) FROM messages m
WHERE m.room_id = ? AND m.seq > ?    -- 내 last_read_seq + 1
  AND m.sender_id = ?                  -- 내 메시지
  AND NOT EXISTS (
    SELECT 1 FROM room_member_read_state r
    WHERE r.room_id = m.room_id
      AND r.user_id != m.sender_id
      AND r.last_read_seq >= m.seq
  );

-- 그룹 — N 명 중 안 읽은 수
SELECT m.seq, (
    SELECT COUNT(*) FROM room_members rm
    WHERE rm.room_id = m.room_id
      AND rm.user_id != m.sender_id
      AND COALESCE(rm.last_read_seq, 0) < m.seq
) AS unread_count
FROM messages m
WHERE m.room_id = ? AND m.created_at >= ?;
```

→ "12명이 안 봤어요" 표시.

---

## 4. 함정

- per-message 의 UNIQUE 없음 → 멀티 디바이스 중복.
- last_read_seq 감소 (스크롤 위) → MAX(old, new) 만 update.
- 카운트 매번 query → Redis cache 5초.

---

## 5. 관련

- [[database|↑ hub]]
- [[../design-decisions/read-receipt-strategy]]
- [[../implementation/read-receipt-impl]]
