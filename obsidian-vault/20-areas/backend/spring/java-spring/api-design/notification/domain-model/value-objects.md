---
title: "notification VOs"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:08:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - value-object
---

# notification VOs

**[[domain-model|↑ hub]]**

---

## 1. VOs

```java
public record NotificationId(String value) { ... }
public record DeviceId(String value) { ... }
public record TemplateKey(String value) {
    // 패턴: SCREAMING_SNAKE — "COMMENT_RECEIVED" / "PAYMENT_APPROVED"
    public TemplateKey {
        require(value.matches("[A-Z][A-Z0-9_]+"), "invalid template key");
    }
}

public record DeviceToken(String value, ChannelType channel) {
    public DeviceToken {
        require(value != null && !value.isBlank());
        // FCM token = ~160 chars, APNs = 64 hex, WebPush = URL
        require(value.length() <= 4096);
    }
}

public record DeliveryResult(
    ChannelType channel,
    boolean success,
    String providerMessageId,    // FCM message id
    String errorCode,
    String errorMessage,
    boolean permanent,
    Instant attemptedAt
) {
    public static DeliveryResult success(ChannelType ch, String msgId, Instant t) {
        return new DeliveryResult(ch, true, msgId, null, null, false, t);
    }
    public static DeliveryResult transient(ChannelType ch, String code, String msg, Instant t) {
        return new DeliveryResult(ch, false, null, code, msg, false, t);
    }
    public static DeliveryResult permanent(ChannelType ch, String code, String msg, Instant t) {
        return new DeliveryResult(ch, false, null, code, msg, true, t);
    }
}

public record NotificationPayload(
    String title,
    String body,
    String deeplink,
    Map<String, String> data,    // FCM data payload
    Priority priority,
    String imageUrl
) { }
```

---

## 2. 관련

- [[domain-model|↑ hub]]
- [[notification-aggregate]]
- [[user-device-aggregate]]
