---
title: "ChannelType enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:22:00+09:00
tags: [backend, java-spring, api-design, notification, enum, channel]
---

# ChannelType enum

**[[enums|↑ hub]]**

```java
public enum ChannelType {
    FCM,         // Android push (Firebase)
    APNS,        // iOS push (Apple)
    WEBPUSH,     // Web (RFC 8030)
    EMAIL,       // SES
    SLACK,       // admin webhook
    IN_APP;      // DB row (사용자 알림 화면)
}
```

---

## 채널별 사용처

| Channel | 대상 | 비용 |
| --- | --- | --- |
| FCM | Android (+ iOS 대안) | 무료 |
| APNS | iOS | $99/년 |
| WEBPUSH | 브라우저 | 무료 |
| EMAIL | 영수증 / 보안 | $0.10 / 1k |
| SLACK | admin only | 무료 |
| IN_APP | 사용자 화면 | DB row |

---

## 관련

- [[enums|↑ hub]]
- [[../design-decisions/channel-selection]]
- [[../prerequisites]]
