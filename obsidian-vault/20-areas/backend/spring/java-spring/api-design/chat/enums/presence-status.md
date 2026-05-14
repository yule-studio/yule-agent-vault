---
title: "PresenceStatus enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:18:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# PresenceStatus enum

**[[enums|↑ hub]]**

```java
public enum PresenceStatus {
    ONLINE,     // 활성 (heartbeat < 1분)
    AWAY,       // idle (heartbeat 1~5분)
    OFFLINE;    // 연결 없음 또는 5분 초과
}
```

```mermaid
stateDiagram-v2
    [*] --> OFFLINE
    OFFLINE --> ONLINE: connect + heartbeat
    ONLINE --> AWAY: 1~5min idle
    AWAY --> ONLINE: 새 heartbeat
    ONLINE --> OFFLINE: disconnect
    AWAY --> OFFLINE: 5min+ idle
```

---

## 관련

- [[enums|↑ hub]]
- [[../design-decisions/presence-strategy]]
