---
title: "rooms 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:32:00+09:00
tags: [backend, java-spring, api-design, chat, database, room]
---

# rooms 테이블

**[[database|↑ hub]]**

---

## 1. Schema

```sql
CREATE TABLE rooms (
    id              CHAR(26) PRIMARY KEY,
    type            VARCHAR(20) NOT NULL,
    name            VARCHAR(200),
    description     TEXT,
    profile_image_url VARCHAR(500),
    max_members     INTEGER NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    creator_id      CHAR(26),
    is_searchable   BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash   VARCHAR(200),
    e2ee_enabled    BOOLEAN NOT NULL DEFAULT FALSE,
    last_message_id CHAR(26),
    last_message_at TIMESTAMPTZ,
    last_seq        BIGINT NOT NULL DEFAULT 0,
    version         BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT chk_room_type CHECK (type IN ('DIRECT','GROUP','OPEN','SECRET'))
);

CREATE INDEX ix_rooms_searchable ON rooms (name, created_at DESC) WHERE is_searchable AND deleted_at IS NULL;
CREATE INDEX ix_rooms_last_message ON rooms (last_message_at DESC);
```

---

## 2. 별도 direct_rooms (pair UNIQUE)

```sql
-- DIRECT room 은 두 user pair 고유
CREATE TABLE direct_rooms (
    room_id     CHAR(26) PRIMARY KEY REFERENCES rooms(id),
    user_a_id   CHAR(26) NOT NULL,
    user_b_id   CHAR(26) NOT NULL,
    CHECK (user_a_id < user_b_id)    -- 정렬 unique
);
CREATE UNIQUE INDEX ux_direct_pair ON direct_rooms (user_a_id, user_b_id);
```

---

## 3. 컬럼 "왜"

- `last_seq` — 메시지 sequence 의 atomic increment 위치 (옵션 A: DB-side).
- `last_message_at` — chat list 정렬 (최신순).
- `member_count` — denormalized (자주 조회).
- `version` — optimistic lock (멤버 join max).
- `is_searchable` — OPEN room 만 true (검색 노출).

---

## 4. 함정

- DIRECT pair 의 UNIQUE 없음 → 같은 두 user의 room 중복.
- last_seq DB atomic UPDATE → 동시 send race → Redis INCR 권장.
- member_count 수동 update → drift.

---

## 5. 관련

- [[database|↑ hub]]
- [[room-members-table]]
- [[../design-decisions/room-types]]
