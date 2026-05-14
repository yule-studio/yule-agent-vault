---
title: "Abuse rate limit"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:34:00+09:00
tags: [backend, java-spring, api-design, notification, security, rate-limit]
---

# Abuse rate limit

**[[security|↑ hub]]**

---

## 1. 시나리오 + 한도

| 시나리오 | 한도 | 조치 |
| --- | --- | --- |
| 단일 사용자 알림 spam | 1000 / day | block + admin alert |
| 같은 IP 의 device 등록 | 50 / day | block (스크립트 의심) |
| 한 사용자 의 token rotation | 20 / day | flag (비정상) |
| outbox 한도 (per user per hour) | 50 (강제 ON 제외) | drop |

자세히: [[../design-decisions/rate-limit]].

---

## 2. 코드 (단일 사용자)

```java
@Scheduled(cron = "0 0 * * * *")    // hourly
public void detectAbuse() {
    var abusers = outboxRepo.findUsersWithCountAbove(1000, Duration.ofDays(1));
    for (var u : abusers) {
        userService.block(u, "notification-abuse");
        slack.alert("user blocked for notification spam", u);
    }
}
```

---

## 3. 관련

- [[security|↑ hub]]
- [[../design-decisions/rate-limit]]
- [[../../rate-limiting|↗ rate-limiting recipe]]
