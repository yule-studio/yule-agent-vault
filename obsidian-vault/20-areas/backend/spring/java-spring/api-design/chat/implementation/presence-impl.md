---
title: "Presence 구현 (heartbeat + typing)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:56:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, presence]
---

# Presence 구현 (heartbeat + typing)

**[[implementation|↑ hub]]**

---

## 1. Presence STOMP

```java
@MessageMapping("/heartbeat")
public void heartbeat(Principal principal) {
    presence.heartbeat(UserId.of(principal.getName()));
}

@MessageMapping("/room/{roomId}/typing")
public void typing(@DestinationVariable String roomId,
                    @Payload TypingRequest req,
                    Principal principal) {
    var user = UserId.of(principal.getName());
    var payload = new TypingEventDto(roomId, user.value(), req.typing());
    ws.convertAndSend("/topic/room/" + roomId + "/typing", payload);
    // 5초 후 자동 false
    scheduler.schedule(() -> {
        ws.convertAndSend("/topic/room/" + roomId + "/typing",
            new TypingEventDto(roomId, user.value(), false));
    }, 5, TimeUnit.SECONDS);
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class PresenceService {

    private final StringRedisTemplate redis;
    private final SimpMessagingTemplate ws;
    private final FriendsService friends;
    private final Clock clock;

    public void heartbeat(UserId user) {
        var key = "presence:" + user.value();
        var current = redis.opsForValue().get(key);
        redis.opsForValue().set(key, "ONLINE", Duration.ofSeconds(90));
        redis.opsForValue().set("lastSeen:" + user.value(),
            clock.now().toString());

        if (!"ONLINE".equals(current)) {
            broadcastToFriends(user, PresenceStatus.ONLINE);
        }
    }

    public void markOnline(UserId user, Instant now) {
        heartbeat(user);
    }

    public void markOffline(UserId user, Instant now) {
        redis.delete("presence:" + user.value());
        redis.opsForValue().set("lastSeen:" + user.value(), now.toString());
        broadcastToFriends(user, PresenceStatus.OFFLINE);
    }

    public boolean isOnline(UserId user) {
        return "ONLINE".equals(redis.opsForValue().get("presence:" + user.value()));
    }

    private void broadcastToFriends(UserId user, PresenceStatus status) {
        var friendIds = friends.friendsOf(user);
        for (var f : friendIds) {
            ws.convertAndSendToUser(f.value(), "/queue/presence",
                new PresenceEventDto(user.value(), status));
        }
    }
}
```

---

## 3. 함정

- heartbeat 매 1초 → 부하 ↑ → 30s 권장.
- typing broadcast 매번 → debounce 1s.
- friend 가 1만+ → broadcast 폭주 → batch.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/presence-strategy]]
- [[../database/presence-cache]]
