---
title: "UserDevice Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:06:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - aggregate
  - device
---

# UserDevice Aggregate

**[[domain-model|↑ hub]]**

---

## 1. Java

```java
public final class UserDevice {
    private final DeviceId id;
    private final UserId userId;
    private final DevicePlatform platform;
    private final ChannelType channelType;
    private DeviceToken token;
    private final String deviceId;
    private String locale;
    private String timezone;
    private boolean active;
    private String invalidReason;
    private Instant lastUsedAt;

    public static UserDevice register(DeviceId id, UserId userId,
                                       DevicePlatform platform, ChannelType ch,
                                       DeviceToken token, String deviceId,
                                       String locale, String timezone,
                                       Instant now) {
        return new UserDevice(id, userId, platform, ch, token, deviceId,
                              locale, timezone, true, null, now);
    }

    public void markUsed(Instant now) { this.lastUsedAt = now; }

    public void markInvalid(String reason) {
        this.active = false;
        this.invalidReason = reason;
    }

    public void rotateToken(DeviceToken newToken, Instant now) {
        this.token = newToken;
        this.active = true;
        this.invalidReason = null;
        this.lastUsedAt = now;
    }

    public void deactivate() { this.active = false; }
}
```

---

## 2. 불변식

- token 1 개 / device (UNIQUE).
- invalid 후 token rotate 시 active=true 복귀.
- logout 시 deactivate 만 (삭제 X — 분석).

---

## 3. 관련

- [[domain-model|↑ hub]]
- [[../database/user-devices-table]]
- [[../design-decisions/device-management]]
