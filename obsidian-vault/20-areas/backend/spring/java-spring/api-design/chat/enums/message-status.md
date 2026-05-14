---
title: "MessageStatus enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:14:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# MessageStatus enum

**[[enums|↑ hub]]**

```java
public enum MessageStatus {
    SENT,        // 서버 INSERT 완료
    DELIVERED,   // 1+ device 도착
    READ,        // 사용자가 읽음
    DELETED,     // 본인 5분 안 hard delete
    HIDDEN;      // 5분 후 soft delete / admin
}
```

```mermaid
stateDiagram-v2
    [*] --> SENT
    SENT --> DELIVERED
    DELIVERED --> READ
    SENT --> DELETED: 본인 5분 안
    SENT --> HIDDEN: 본인 5분 후 / admin
    DELIVERED --> HIDDEN
    READ --> HIDDEN
```

---

## 관련

- [[enums|↑ hub]]
- [[../domain-model/message-aggregate]]
