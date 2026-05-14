---
title: "room_events 테이블 — 입장/퇴장/방장변경 audit"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:46:00+09:00
tags: [backend, java-spring, api-design, chat, database, audit]
---

# room_events 테이블 — 입장/퇴장/방장변경 audit

**[[database|↑ hub]]**

---

## Schema

```sql
CREATE TABLE room_events (
    id           CHAR(26) PRIMARY KEY,
    room_id      CHAR(26) NOT NULL,
    actor_id     CHAR(26),                           -- 행위자 (system 시 NULL)
    target_id    CHAR(26),                           -- 영향 받은 user
    event_type   VARCHAR(50) NOT NULL,                -- MEMBER_JOIN / LEAVE / KICK / OWNER_CHANGE / ROOM_UPDATE
    metadata     JSONB,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_re_room ON room_events (room_id, created_at DESC);
CREATE INDEX ix_re_actor ON room_events (actor_id, created_at DESC);
```

---

## 시스템 메시지로 표시

- 카톡 "OO님이 들어왔어요" / "OO님이 나갔어요" = `messages.type=SYSTEM` + content rendered.
- room_events 는 audit (admin 추적용) — 사용자 화면 X.

---

## 관련

- [[database|↑ hub]]
- [[../security/audit-logging]]
