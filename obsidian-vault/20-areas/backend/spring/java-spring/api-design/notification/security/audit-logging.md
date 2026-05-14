---
title: "Audit logging (5년)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:38:00+09:00
tags: [backend, java-spring, api-design, notification, security, audit]
---

# Audit logging (5년)

**[[security|↑ hub]]**

---

## 1. 무엇 audit

| Event | 보유 |
| --- | --- |
| 발송 시도 + 결과 (notification_outbox 영구 후 cold storage) | 5년 |
| device 등록 / 삭제 | 5년 |
| preference 변경 | 5년 |
| DLQ admin replay | 5년 |
| abuse block | 5년 |

---

## 2. Schema (별도 audit table)

```sql
CREATE TABLE notification_audit_log (
    id              CHAR(26) PRIMARY KEY,
    event_type      VARCHAR(50) NOT NULL,
    user_id         CHAR(26),
    actor_id        CHAR(26),
    actor_role      VARCHAR(20),
    notification_type VARCHAR(50),
    channel         VARCHAR(20),
    result          VARCHAR(20),
    metadata        JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_audit_user ON notification_audit_log (user_id, occurred_at DESC);
CREATE INDEX ix_audit_event ON notification_audit_log (event_type, occurred_at DESC);
```

---

## 3. Cleanup

```sql
-- 5년 지난 audit cold storage 로 archive 후 DELETE
DELETE FROM notification_audit_log
WHERE occurred_at < now() - INTERVAL '5 years';
```

---

## 관련

- [[security|↑ hub]]
- [[../../signup/security/audit-logging|↗ signup audit]]
- [[../../product/security/audit-logging|↗ product audit]]
