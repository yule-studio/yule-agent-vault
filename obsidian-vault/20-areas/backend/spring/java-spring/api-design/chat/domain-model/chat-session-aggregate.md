---
title: "ChatSession Aggregate (WebSocket session)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:56:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, aggregate, session]
---

# ChatSession Aggregate (WebSocket session)

**[[domain-model|↑ hub]]**

---

```java
public final class ChatSession {
    private final SessionId id;          // STOMP session id
    private final UserId userId;
    private final DeviceId deviceId;     // FE 부여 UUID
    private final String nodeId;          // chat-1 / chat-2
    private final Instant connectedAt;
    private Instant lastHeartbeatAt;
    private Instant disconnectedAt;
    private final String ipAddress;
    private final String userAgent;

    public static ChatSession connect(SessionId id, UserId user, DeviceId device,
                                       String node, String ip, String ua, Instant now) {
        return new ChatSession(id, user, device, node, now, now, null, ip, ua);
    }

    public void heartbeat(Instant now) { this.lastHeartbeatAt = now; }

    public void disconnect(Instant now) { this.disconnectedAt = now; }

    public boolean isActive(Instant now) {
        return disconnectedAt == null
            && Duration.between(lastHeartbeatAt, now).toMinutes() < 5;
    }
}
```

---

## 저장 위치

- Redis (primary) — 짧은 lifetime + 빠른 lookup.
- DB (옵션) — audit / 분쟁.

자세히: [[../database/chat-sessions-table]].

---

## 관련

- [[domain-model|↑ hub]]
- [[../security/websocket-auth]]
- [[../implementation/connect-auth-impl]]
