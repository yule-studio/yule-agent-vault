---
title: "InApp Channel 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:56:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, in-app]
---

# InApp Channel 구현

**[[implementation|↑ hub]]**

---

## 1. 코드

```java
@Component
@RequiredArgsConstructor
public class InAppChannel implements NotificationChannel {

    private final NotificationRepository repo;
    private final IdGenerator ids;
    private final Clock clock;

    public ChannelType type() { return ChannelType.IN_APP; }
    public DevicePlatform[] supportedPlatforms() { return new DevicePlatform[] {}; }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        // device 없음 — user_id 기반
        var userId = UserId.of(payload.data().get("userId"));

        repo.save(new InAppNotification(
            ids.next(),
            userId,
            payload.data().get("notificationType"),
            payload.title(),
            payload.body(),
            payload.deeplink(),
            payload.data(),
            payload.imageUrl(),
            null,    // read_at
            clock.now()));

        return DeliveryResult.success(ChannelType.IN_APP, "in-app-row", clock.now());
    }
}
```

---

## 2. API

```http
GET    /api/v1/me/notifications?cursor=...
PATCH  /api/v1/me/notifications/{id}/read
PATCH  /api/v1/me/notifications/read-all
GET    /api/v1/me/notifications/unread-count
DELETE /api/v1/me/notifications/{id}
```

---

## 3. 사용자별 unread count

```java
// Redis cache 5분
public int unreadCount(UserId user) {
    var key = "notif:unread:" + user.value();
    var cached = redis.opsForValue().get(key);
    if (cached != null) return Integer.parseInt(cached);

    int count = repo.countUnreadByUser(user);
    redis.opsForValue().set(key, String.valueOf(count), Duration.ofMinutes(5));
    return count;
}

// 읽음 처리 시 cache invalidate
public void markRead(NotificationId id, UserId user) {
    repo.markRead(id, user, clock.now());
    redis.delete("notif:unread:" + user.value());
}
```

---

## 4. 관련

- [[implementation|↑ hub]]
- [[../database/notifications-table]]
