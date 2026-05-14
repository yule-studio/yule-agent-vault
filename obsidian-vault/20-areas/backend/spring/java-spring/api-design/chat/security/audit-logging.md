---
title: "Audit — admin / abuse / 5년 보유"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:38:00+09:00
tags: [backend, java-spring, api-design, chat, security, audit]
---

# Audit — admin / abuse / 5년 보유

**[[security|↑ hub]]**

---

## 1. 무엇 audit (5년)

| Event | 보유 |
| --- | --- |
| admin 메시지 hide | 5년 |
| 그룹 owner 변경 / 강퇴 | 5년 |
| 사용자 block / unblock | 5년 |
| 사용자 신고 → 처리 | 5년 |
| WebSocket connect / disconnect (보안 audit) | 1년 |

---

## 2. Schema

```sql
CREATE TABLE chat_audit_log (
    id           CHAR(26) PRIMARY KEY,
    event_type   VARCHAR(50) NOT NULL,
    actor_id     CHAR(26),
    target_id    CHAR(26),
    room_id      CHAR(26),
    message_id   CHAR(26),
    metadata     JSONB,
    ip_address   INET,
    user_agent   TEXT,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_cal_room ON chat_audit_log (room_id, occurred_at DESC);
CREATE INDEX ix_cal_actor ON chat_audit_log (actor_id, occurred_at DESC);
```

---

## 관련

- [[security|↑ hub]]
- [[../database/room-events-table]]
- [[../../signup/security/audit-logging|↗ signup audit]]
