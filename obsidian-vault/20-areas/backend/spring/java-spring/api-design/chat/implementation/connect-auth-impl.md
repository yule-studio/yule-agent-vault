---
title: "Connect auth 구현 ★ (StompAuthInterceptor + session 등록)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:44:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, auth]
---

# Connect auth 구현 ★ (StompAuthInterceptor + session 등록)

**[[implementation|↑ hub]]**

> 코드 자세히: [[../security/websocket-auth]].

---

## 1. Interceptor

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class StompAuthInterceptor implements ChannelInterceptor {

    private final JwtValidator jwt;
    private final ChatSessionRepository sessions;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        var acc = StompHeaderAccessor.wrap(message);

        switch (acc.getCommand()) {
            case CONNECT -> handleConnect(acc);
            case SUBSCRIBE -> validateSubscribe(acc);
            case DISCONNECT -> handleDisconnect(acc);
            default -> {}
        }
        return message;
    }

    private void handleConnect(StompHeaderAccessor acc) {
        var token = extractToken(acc);
        var claims = jwt.validate(token);
        var user = UserId.of(claims.getSubject());

        acc.setUser(new UsernamePasswordAuthenticationToken(user.value(), null, List.of()));

        var sessionId = SessionId.of(acc.getSessionId());
        var deviceId = DeviceId.of(acc.getFirstNativeHeader("Device-Id"));

        sessions.register(ChatSession.connect(
            sessionId, user, deviceId,
            System.getProperty("nodeId", "default"),
            (String) acc.getSessionAttributes().getOrDefault("ip", "?"),
            (String) acc.getSessionAttributes().getOrDefault("ua", "?"),
            clock.now()));

        events.publishEvent(new ChatSessionConnected(sessionId, user, clock.now()));
    }

    private void validateSubscribe(StompHeaderAccessor acc) {
        var destination = acc.getDestination();    // "/topic/room/abc"
        var user = acc.getUser();
        if (user == null) throw new MessageDeliveryException("unauthenticated");

        if (destination != null && destination.startsWith("/topic/room/")) {
            var roomId = RoomId.of(destination.substring("/topic/room/".length()));
            // room member 검증 — RoomService 호출
        }
    }

    private void handleDisconnect(StompHeaderAccessor acc) {
        var sessionId = SessionId.of(acc.getSessionId());
        sessions.unregister(sessionId);
        events.publishEvent(new ChatSessionDisconnected(sessionId, clock.now()));
    }

    private String extractToken(StompHeaderAccessor acc) {
        var auth = acc.getFirstNativeHeader("Authorization");
        if (auth == null || !auth.startsWith("Bearer "))
            throw new MessageDeliveryException("missing token");
        return auth.substring(7);
    }
}
```

---

## 2. Event listener (Spring native)

```java
@Component
@RequiredArgsConstructor
public class WebSocketEventListener {

    private final PresenceService presence;
    private final Clock clock;

    @EventListener
    public void onConnect(SessionConnectedEvent event) {
        var user = UserId.of(event.getUser().getName());
        presence.markOnline(user, clock.now());
    }

    @EventListener
    public void onDisconnect(SessionDisconnectEvent event) {
        var user = UserId.of(event.getUser().getName());
        presence.markOffline(user, clock.now());
    }
}
```

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../security/websocket-auth]]
- [[../security/csrf-websocket]]
- [[../database/chat-sessions-table]]
