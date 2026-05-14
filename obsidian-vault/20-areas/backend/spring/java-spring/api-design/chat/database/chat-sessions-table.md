---
title: "chat_sessions (Redis 또는 DB)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:48:00+09:00
tags: [backend, java-spring, api-design, chat, database, session]
---

# chat_sessions (Redis 또는 DB)

**[[database|↑ hub]]**

> WebSocket session 정보 — 본 vault 는 **Redis** (DB X — 너무 짧은 lifetime).

---

## 1. Redis

```
HSET chat:session:{sessionId}
  user_id   abc123
  node_id   chat-1
  connected_at 2026-05-14T15:00:00Z
EXPIRE chat:session:{sessionId} 1h    # heartbeat 마다 갱신

SADD user:{userId}:sessions {sessionId}
EXPIRE user:{userId}:sessions 1h
```

→ DISCONNECT 시 cleanup.

---

## 2. 옵션 — DB 보관 (audit)

```sql
CREATE TABLE chat_session_audit (
    id              CHAR(26) PRIMARY KEY,
    session_id      VARCHAR(100) NOT NULL,
    user_id         CHAR(26) NOT NULL,
    node_id         VARCHAR(50),
    device_info     JSONB,
    ip_address      INET,
    connected_at    TIMESTAMPTZ NOT NULL,
    disconnected_at TIMESTAMPTZ,
    duration_seconds INTEGER
);

CREATE INDEX ix_csa_user ON chat_session_audit (user_id, connected_at DESC);
```

→ "사용자가 어떤 device 에서 언제 접속" — security audit.

---

## 3. 관련

- [[database|↑ hub]]
- [[../security/websocket-auth]]
- [[../implementation/connect-auth-impl]]
