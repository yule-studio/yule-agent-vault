---
title: "presence (Redis only, DB X)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:42:00+09:00
tags: [backend, java-spring, api-design, chat, database, presence, redis]
---

# presence (Redis only, DB X)

**[[database|↑ hub]]**

---

## 1. Redis schema

```
SET presence:{userId} ONLINE EX 90       # state + 90s TTL
SET lastSeen:{userId} 2026-05-14T15:00:00Z   # 영구 (사용자 마지막 활동 시각)

SADD user:{userId}:sessions {sessionId}     # multi-device 의 session list
EXPIRE user:{userId}:sessions 1h
```

---

## 2. 왜 Redis 만 (DB X)

- 사용자 1만 명 × 30s heartbeat = 분당 2만 write → DB 불가능.
- in-memory 빠름.
- 영속 불필요 (휘발성 정보).

---

## 3. lastSeen 만 DB (옵션)

```sql
CREATE TABLE user_last_seen (
    user_id      CHAR(26) PRIMARY KEY,
    last_seen_at TIMESTAMPTZ NOT NULL,
    show_to      VARCHAR(20) NOT NULL DEFAULT 'FRIENDS'  -- PUBLIC / FRIENDS / NONE
);
```

→ "마지막 접속" 표시용 — heartbeat 시 매번 X (분당 1회 batch update).

---

## 4. 함정

- 모든 presence DB → 부하 폭주.
- Redis 1 노드 SPOF — cluster.
- heartbeat 매 1초 → 부하 ↑ → 30s 권장.

---

## 관련

- [[database|↑ hub]]
- [[../design-decisions/presence-strategy]]
- [[../implementation/presence-impl]]
