---
title: "notification_dlq 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - dlq
---

# notification_dlq 테이블

**[[database|↑ hub]]**

> max retry 초과한 outbox row 의 별도 보관 — admin replay 가능.

---

## 1. Schema

```sql
CREATE TABLE notification_dlq (
    id                CHAR(26) PRIMARY KEY,
    original_outbox_id CHAR(26),
    event_id          VARCHAR(100) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    user_id           CHAR(26) NOT NULL,
    template_key      VARCHAR(100),
    variables         JSONB NOT NULL,
    channel_types     TEXT[] NOT NULL,
    failure_reason    TEXT NOT NULL,
    attempts          INTEGER NOT NULL,
    last_error        TEXT,
    moved_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    replayed_at       TIMESTAMPTZ,
    replayed_by       CHAR(26),
    resolved          BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX ix_dlq_user_moved ON notification_dlq (user_id, moved_at DESC);
CREATE INDEX ix_dlq_unresolved ON notification_dlq (moved_at DESC) WHERE NOT resolved;
```

---

## 2. Admin replay API

```http
GET    /api/v1/admin/notifications/dlq?cursor=...
POST   /api/v1/admin/notifications/dlq/{id}/replay
POST   /api/v1/admin/notifications/dlq/{id}/resolve   # 무시 처리
```

```java
@Transactional
public void replay(String dlqId, UserId admin) {
    var dlq = repo.findById(dlqId).orElseThrow();
    outbox.save(NotificationOutboxRow.replayFrom(dlq));
    dlq.markReplayed(admin, clock.now());
    repo.save(dlq);
}
```

---

## 3. Cleanup

```sql
-- resolved + 90일 지난 DLQ
DELETE FROM notification_dlq
WHERE resolved = TRUE AND moved_at < now() - INTERVAL '90 days';
```

→ unresolved 는 영구 (분쟁 대비).

---

## 4. 관련

- [[database|↑ hub]]
- [[notification-outbox-table]]
- [[../operations/dlq-replay]]
- [[../design-decisions/retry-strategy]]
