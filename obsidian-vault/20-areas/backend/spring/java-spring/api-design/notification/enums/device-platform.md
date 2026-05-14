---
title: "DevicePlatform enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:24:00+09:00
tags: [backend, java-spring, api-design, notification, enum, device]
---

# DevicePlatform enum

**[[enums|↑ hub]]**

```java
public enum DevicePlatform {
    ANDROID,
    IOS,
    WEB,
    DESKTOP;
}
```

| Platform | 주 채널 |
| --- | --- |
| ANDROID | FCM |
| IOS | APNS (FCM 도 가능) |
| WEB | WEBPUSH |
| DESKTOP | WEBPUSH (Electron) 또는 SLACK |

---

## 관련

- [[enums|↑ hub]]
- [[channel-type]]
