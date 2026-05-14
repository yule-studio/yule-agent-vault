---
title: "blocks 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:44:00+09:00
tags: [backend, java-spring, api-design, chat, database, block]
---

# blocks 테이블

**[[database|↑ hub]]**

---

## Schema

```sql
CREATE TABLE blocks (
    blocker_id  CHAR(26) NOT NULL,
    blocked_id  CHAR(26) NOT NULL,
    reason      VARCHAR(500),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id),
    CHECK (blocker_id != blocked_id)
);

CREATE INDEX ix_blocks_blocked ON blocks (blocked_id);
```

---

## Block 확인 (filter)

```java
// 사용자 A 의 차단 list (Redis cache 5분)
public Set<UserId> blockedBy(UserId user) {
    var key = "blocks:" + user.value();
    var cached = redis.opsForSet().members(key);
    if (cached != null && !cached.isEmpty()) return parse(cached);

    var fromDb = repo.findBlockedByUser(user);
    redis.opsForSet().add(key, ...);
    redis.expire(key, Duration.ofMinutes(5));
    return fromDb;
}
```

자세히: [[../design-decisions/moderation-blocking]] · [[../security/blocked-user-filter]].

---

## 관련

- [[database|↑ hub]]
- [[../security/blocked-user-filter]]
