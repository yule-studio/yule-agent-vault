---
title: "NotificationPriority enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:26:00+09:00
tags: [backend, java-spring, api-design, notification, enum]
---

# NotificationPriority enum

**[[enums|↑ hub]]**

```java
public enum NotificationPriority {
    LOW,         // 마케팅 / 추천
    NORMAL,      // 좋아요 / 댓글
    HIGH,        // 결제 / 보안
    CRITICAL;    // fraud / 즉시 사용자 보호
}
```

---

## Worker pickup 우선순위

```sql
SELECT * FROM notification_outbox
WHERE status IN ('PENDING', 'PROCESSING')
  AND next_attempt_at <= now()
ORDER BY priority DESC, created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

→ CRITICAL → HIGH → NORMAL → LOW.

---

## FCM / APNs priority 매핑

| 우리 | FCM | APNs |
| --- | --- | --- |
| LOW | priority=normal | apns-priority=5 |
| NORMAL | priority=normal | apns-priority=10 |
| HIGH | priority=high | apns-priority=10 |
| CRITICAL | priority=high + content_available | apns-priority=10 + apns-push-type=alert |

---

## 관련

- [[enums|↑ hub]]
- [[../design-decisions/ordering]]
- [[../implementation/outbox-worker-impl]]
