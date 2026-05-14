---
title: "Rate limit — spam 방어"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - rate-limit
---

# Rate limit — spam 방어

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

| 한도 | 적용 |
| --- | --- |
| **50 / hour / user** | 일반 type — 초과 시 throttle (집계 또는 drop) |
| **20 / hour / user / type** | 같은 type 폭주 (예: 좋아요 spam) |
| **무한** | 강제 ON (보안 / 결제) |
| **Burst 10/10s** | 자동 집계 ([[dedup-strategy#4]]) |

---

## 2. 왜

- 인기 글의 좋아요 분당 100 → 분당 100 알림 → 사용자 OFF → 다른 알림 무시.
- → rate limit + 집계 = balance.

---

## 3. Redis 구현

```java
@Service
public class NotificationRateLimiter {

    private final StringRedisTemplate redis;

    public boolean tryConsume(UserId user, NotificationType type) {
        if (type.enforced()) return true;   // 강제 ON skip

        // per-user (hourly)
        var userKey = "rl:notif:user:%s".formatted(user.value());
        var userCount = redis.opsForValue().increment(userKey);
        if (userCount == 1) redis.expire(userKey, Duration.ofHours(1));
        if (userCount > 50) return false;

        // per-user-per-type
        var typeKey = "rl:notif:user:%s:type:%s".formatted(user.value(), type.name());
        var typeCount = redis.opsForValue().increment(typeKey);
        if (typeCount == 1) redis.expire(typeKey, Duration.ofHours(1));
        if (typeCount > 20) return false;

        return true;
    }
}
```

→ 초과 시 outbox INSERT skip + log.

---

## 4. 함정

### 함정 1 — 강제 ON 도 limit
보안 alert drop.
→ enforced skip.

### 함정 2 — TTL 없음
무한 누적.

### 함정 3 — drop 만 (집계 X)
중요한 사용자가 봐야 할 알림 손실.
→ 집계 + drop 조합.

---

## 5. 관련

- [[design-decisions|↑ hub]]
- [[batch-aggregation]]
- [[dedup-strategy]]
- [[../../rate-limiting|↗ rate-limiting recipe]]
