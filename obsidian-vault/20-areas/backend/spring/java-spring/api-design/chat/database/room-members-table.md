---
title: "room_members 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:34:00+09:00
tags: [backend, java-spring, api-design, chat, database, member]
---

# room_members 테이블

**[[database|↑ hub]]**

---

## 1. Schema

```sql
CREATE TABLE room_members (
    room_id        CHAR(26) NOT NULL REFERENCES rooms(id),
    user_id        CHAR(26) NOT NULL,
    role           VARCHAR(20) NOT NULL DEFAULT 'MEMBER',
    nickname       VARCHAR(50),                  -- 그룹 내 닉네임
    joined_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_read_seq  BIGINT NOT NULL DEFAULT 0,
    last_read_at   TIMESTAMPTZ,
    muted_until    TIMESTAMPTZ,                   -- 알림 음소거
    pinned         BOOLEAN NOT NULL DEFAULT FALSE,
    left_at        TIMESTAMPTZ,
    PRIMARY KEY (room_id, user_id),

    CONSTRAINT chk_role CHECK (role IN ('OWNER','ADMIN','MEMBER'))
);

CREATE INDEX ix_rm_user ON room_members (user_id, last_read_at DESC) WHERE left_at IS NULL;
CREATE INDEX ix_rm_room ON room_members (room_id) WHERE left_at IS NULL;
```

---

## 2. 컬럼 "왜"

- `role` — 그룹의 OWNER (방장) / ADMIN / MEMBER.
- `nickname` — 그룹 안 닉네임 (사용자 본명과 별도).
- `last_read_seq` — [[../design-decisions/read-receipt-strategy#22]] last_read_seq 의 PK.
- `muted_until` — 알림 음소거 (1시간 / 1일 / 영구).
- `pinned` — 사용자의 즐겨찾기 room.
- `left_at` — soft delete (퇴장 후 메시지 history 보존).

---

## 3. 사용자 chat list query

```sql
SELECT r.*, m.last_read_seq, r.last_seq, r.last_message_at
FROM room_members m
JOIN rooms r ON r.id = m.room_id
WHERE m.user_id = ?
  AND m.left_at IS NULL
  AND r.deleted_at IS NULL
ORDER BY m.pinned DESC, r.last_message_at DESC NULLS LAST
LIMIT 50;
```

---

## 4. 함정

- max_member 초과 join — application 검증 + room.version.
- left_at 후 다시 join → INSERT 또는 UPDATE — UPSERT.
- last_read_seq 감소 (스크롤 업) → MAX(old, new).

---

## 5. 관련

- [[database|↑ hub]]
- [[rooms-table]]
- [[../design-decisions/read-receipt-strategy]]
- [[../enums/member-role]]
