---
title: "NotificationType enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:18:00+09:00
tags: [backend, java-spring, api-design, notification, enum]
---

# NotificationType enum

**[[enums|↑ hub]]**

---

## 1. 값

```java
public enum NotificationType {
    // 결제 (강제 ON)
    PAYMENT_APPROVED(true),
    PAYMENT_FAILED(true),
    REFUND_PROCESSED(true),

    // 보안 (강제 ON)
    SECURITY_LOGIN_NEW_DEVICE(true),
    SECURITY_PASSWORD_CHANGED(true),
    SECURITY_2FA_ENABLED(true),

    // 모더 (강제 ON)
    MODERATION_HIDDEN(true),
    MODERATION_RESTORED(true),

    // 커뮤니티
    COMMENT_RECEIVED(false),
    REPLY_RECEIVED(false),
    LIKE_RECEIVED(false),
    MENTION(false),

    // chat
    CHAT_MESSAGE(false),
    CHAT_INVITED(false),

    // 마케팅
    MARKETING_PROMOTION(false),
    MARKETING_NEWSLETTER(false),

    // admin
    ADMIN_FRAUD_ALERT(false),    // → Slack
    ADMIN_5XX_SPIKE(false);

    private final boolean enforced;
    NotificationType(boolean enforced) { this.enforced = enforced; }
    public boolean enforced() { return enforced; }
}
```

---

## 2. 관련

- [[enums|↑ hub]]
- [[../design-decisions/channel-selection]]
- [[../design-decisions/user-preferences]]
