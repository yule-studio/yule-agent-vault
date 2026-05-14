---
title: "Repository Ports + NotificationChannel port"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:12:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - port
---

# Repository Ports + NotificationChannel port

**[[domain-model|↑ hub]]**

---

## 1. Repositories

```java
public interface NotificationOutboxRepository {
    Notification save(Notification n);
    Optional<Notification> findById(NotificationId id);
    List<Notification> findPendingForUpdateSkipLocked(int limit, Instant now);
    int deactivateOldSent(Duration olderThan);   // cleanup
}

public interface UserDeviceRepository {
    UserDevice save(UserDevice d);
    Optional<UserDevice> findByToken(String token);
    List<UserDevice> findActiveByUser(UserId user, ChannelType channel);
    int deactivateInactiveBefore(Instant threshold);
}

public interface UserNotificationPreferenceRepository {
    UserNotificationPreference save(UserNotificationPreference p);
    Optional<UserNotificationPreference> findByUserId(UserId user);
}

public interface NotificationTemplateRepository {
    Optional<NotificationTemplate> findByKeyAndChannelAndLocale(
        TemplateKey key, ChannelType ch, String locale);
}

public interface NotificationDlqRepository {
    NotificationDlqRow save(NotificationDlqRow row);
    List<NotificationDlqRow> findUnresolved(Pageable page);
}
```

---

## 2. NotificationChannel port ★

```java
public interface NotificationChannel {
    ChannelType type();
    DevicePlatform[] supportedPlatforms();
    DeliveryResult send(NotificationPayload payload, UserDevice device);
}
```

→ Infra layer 가 구현:
- `FcmChannel` (Firebase Admin SDK)
- `ApnsChannel` (Pushy / HTTP2)
- `WebPushChannel` (RFC 8030)
- `SesEmailChannel` (AWS SDK)
- `SlackChannel` (Incoming webhook)
- `InAppChannel` (DB INSERT — 다른 채널과 같은 interface)

---

## 3. 관련

- [[domain-model|↑ hub]]
- [[../architecture]]
- [[../implementation/channel-router-impl]]
