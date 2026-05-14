---
title: "Rate limit — 메시지 spam / 도배"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:34:00+09:00
tags: [backend, java-spring, api-design, chat, security, rate-limit]
---

# Rate limit — 메시지 spam / 도배

**[[security|↑ hub]]**

---

## 1. 본 vault

| 한도 | 적용 |
| --- | --- |
| 10 / s / user | 일반 (Redis 토큰버킷) |
| 30 / 10s / user / room | 같은 room 도배 |
| 1000 / 1h / user | 전체 spam 한도 → block alert |

---

## 2. Redis 토큰 버킷

```java
@Service
public class MessageRateLimiter {

    private final StringRedisTemplate redis;

    public boolean tryConsume(UserId user, RoomId room) {
        var userKey = "rl:msg:user:" + user.value();
        var roomKey = "rl:msg:user:%s:room:%s".formatted(user.value(), room.value());

        var userCount = redis.opsForValue().increment(userKey);
        if (userCount == 1) redis.expire(userKey, Duration.ofSeconds(1));
        if (userCount > 10) return false;

        var roomCount = redis.opsForValue().increment(roomKey);
        if (roomCount == 1) redis.expire(roomKey, Duration.ofSeconds(10));
        if (roomCount > 30) return false;

        return true;
    }
}
```

---

## 3. STOMP 에서 사용

```java
@MessageMapping("/room/{roomId}/send")
public void send(...) {
    if (!rateLimiter.tryConsume(userId, roomId)) {
        // STOMP ERROR frame 으로 reject
        throw new RateLimitedException();
    }
    // ...
}
```

---

## 4. 함정

1. **TTL 없음** → key 영구 누적.
2. **WebSocket 만 검증** → REST 경로 bypass.
3. **Redis fail 시 fail-open** → spam 그대로.
   → fail-closed (보수적).

---

## 관련

- [[security|↑ hub]]
- [[../../rate-limiting|↗ rate-limiting recipe]]
