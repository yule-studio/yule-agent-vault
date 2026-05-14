---
title: "Presence Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:58:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, aggregate, presence]
---

# Presence Aggregate

**[[domain-model|↑ hub]]**

---

```java
public final class Presence {
    private final UserId userId;
    private PresenceStatus status;
    private Instant lastSeenAt;
    private final ShowLastSeenTo visibility;     // PUBLIC / FRIENDS / NONE

    public static Presence online(UserId user, Instant now) {
        return new Presence(user, PresenceStatus.ONLINE, now, ShowLastSeenTo.FRIENDS);
    }

    public void heartbeat(Instant now) {
        this.status = PresenceStatus.ONLINE;
        this.lastSeenAt = now;
    }

    public void markAway(Instant now) {
        if (status == PresenceStatus.ONLINE) this.status = PresenceStatus.AWAY;
    }

    public void disconnect(Instant now) {
        this.status = PresenceStatus.OFFLINE;
        this.lastSeenAt = now;
    }

    public boolean canSee(UserId viewer, FriendChecker friends) {
        return switch (visibility) {
            case PUBLIC -> true;
            case FRIENDS -> friends.isFriend(userId, viewer);
            case NONE -> false;
        };
    }
}
```

---

## 저장

- Redis (state + TTL).
- DB X — 너무 자주 변경.

자세히: [[../database/presence-cache]].

---

## 관련

- [[domain-model|↑ hub]]
- [[../design-decisions/presence-strategy]]
- [[../enums/presence-status]]
