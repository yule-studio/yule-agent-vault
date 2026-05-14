---
title: "NotificationStatus enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:20:00+09:00
tags: [backend, java-spring, api-design, notification, enum]
---

# NotificationStatus enum

**[[enums|↑ hub]]**

```java
public enum NotificationStatus {
    PENDING,
    PROCESSING,
    SENT,
    FAILED,
    DLQ;
}
```

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> PROCESSING: worker pickup
    PROCESSING --> SENT
    PROCESSING --> PENDING: transient retry
    PROCESSING --> FAILED: permanent
    PENDING --> DLQ: max attempts
    FAILED --> DLQ
```

---

## 관련

- [[enums|↑ hub]]
- [[../domain-model/notification-aggregate]]
- [[../design-decisions/retry-strategy]]
