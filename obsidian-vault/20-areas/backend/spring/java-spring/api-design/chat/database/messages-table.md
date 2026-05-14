---
title: "messages 테이블 ★ (partitioned by month)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:36:00+09:00
tags: [backend, java-spring, api-design, chat, database, message]
---

# messages 테이블 ★ (partitioned by month)

**[[database|↑ hub]]**

---

## 1. Schema

```sql
CREATE TABLE messages (
    id              CHAR(26) NOT NULL,
    room_id         CHAR(26) NOT NULL,
    sender_id       CHAR(26) NOT NULL,
    seq             BIGINT NOT NULL,                -- room 별 sequence
    type            VARCHAR(20) NOT NULL,
    content         TEXT,
    metadata        JSONB,                           -- type 별 데이터
    reply_to_id     CHAR(26),                        -- 답글
    client_message_id VARCHAR(50),                   -- idempotency (옵션)
    status          VARCHAR(20) NOT NULL DEFAULT 'SENT',
    deleted_at      TIMESTAMPTZ,
    edited_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (id, created_at),
    CONSTRAINT chk_msg_status CHECK
        (status IN ('SENT','DELIVERED','READ','DELETED','HIDDEN')),
    CONSTRAINT chk_msg_type CHECK
        (type IN ('TEXT','IMAGE','VIDEO','AUDIO','FILE','EMOJI','SYSTEM'))
) PARTITION BY RANGE (created_at);

CREATE TABLE messages_2026_05 PARTITION OF messages
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX ix_msg_room_seq ON messages_2026_05 (room_id, seq DESC);
CREATE INDEX ix_msg_room_created ON messages_2026_05 (room_id, created_at DESC);
CREATE UNIQUE INDEX ux_msg_room_seq ON messages_2026_05 (room_id, seq);
CREATE UNIQUE INDEX ux_msg_client ON messages_2026_05 (sender_id, client_message_id)
    WHERE client_message_id IS NOT NULL;
```

---

## 2. 컬럼 "왜"

- `seq` — 순서 + UNIQUE [[../design-decisions/message-ordering]].
- `client_message_id` — FE 부여 UUID — idempotency (더블 클릭 방어).
- `metadata JSONB` — IMAGE 의 dimensions / VIDEO 의 duration / FILE 의 mime.
- `reply_to_id` — 답글 (인용).
- `status` — SENT/DELIVERED/READ/DELETED/HIDDEN.
- `deleted_at` — soft delete (5분 후) / hard delete (5분 안).
- `edited_at` — 메시지 편집 (5분 안).

---

## 3. cursor pagination

```sql
-- 위로 (옛 메시지)
SELECT * FROM messages
WHERE room_id = ?
  AND (seq < ? OR (seq = ? AND id < ?))
  AND created_at >= ?    -- partition prune hint
ORDER BY seq DESC, id DESC
LIMIT 30;
```

---

## 4. 함정

- partition 안 함 → 단일 테이블 100억 row.
- (room_id, seq) UNIQUE 없음 → 순서 race 시 중복.
- soft delete 후에도 검색 노출.
- partition prune 위해 `created_at` range 필요.

---

## 5. 관련

- [[database|↑ hub]]
- [[../design-decisions/message-storage]]
- [[../design-decisions/message-ordering]]
- [[message-reads-table]]
