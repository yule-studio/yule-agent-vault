---
title: "WebSocket auth — CONNECT frame + JWT ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:24:00+09:00
tags: [backend, java-spring, api-design, chat, security, websocket, auth]
---

# WebSocket auth — CONNECT frame + JWT ★

**[[security|↑ hub]]**

---

## 1. 흐름

```
[Client]
  STOMP CONNECT
    nativeHeaders["Authorization"] = "Bearer <JWT>"
  ↓
[StompAuthInterceptor]
  parse JWT → user_id 추출
  검증 (expiry / signature / blacklist)
  set Principal
  ↓
[Spring SimpUserRegistry]
  user_id 기준 session 등록
  ↓
SUBSCRIBE / SEND 시 Principal 사용
```

---

## 2. 코드 (StompAuthInterceptor)

```java
@Component
@RequiredArgsConstructor
public class StompAuthInterceptor implements ChannelInterceptor {

    private final JwtValidator jwtValidator;
    private final ChatSessionRepository sessions;
    private final Clock clock;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        var accessor = StompHeaderAccessor.wrap(message);

        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            var auth = accessor.getFirstNativeHeader("Authorization");
            if (auth == null || !auth.startsWith("Bearer ")) {
                throw new MessageDeliveryException("missing token");
            }
            var token = auth.substring(7);
            var claims = jwtValidator.validate(token);   // 만료 / 서명 / blacklist

            var principal = new UsernamePasswordAuthenticationToken(
                claims.getSubject(), null, List.of());
            accessor.setUser(principal);

            // session 등록
            sessions.register(ChatSession.connect(
                SessionId.of(accessor.getSessionId()),
                UserId.of(claims.getSubject()),
                /* deviceId */ DeviceId.of(accessor.getFirstNativeHeader("Device-Id")),
                System.getProperty("nodeId"),
                accessor.getSessionAttributes().get("ip").toString(),
                accessor.getSessionAttributes().get("ua").toString(),
                clock.now()));
        }
        return message;
    }
}
```

---

## 3. Spring config

```java
@Override
public void configureClientInboundChannel(ChannelRegistration registration) {
    registration.interceptors(stompAuthInterceptor);
}
```

---

## 4. SUBSCRIBE / SEND 시 검증

```java
@MessageMapping("/room/{roomId}/send")
public void send(@DestinationVariable String roomId,
                 @Payload SendMessageRequest req,
                 Principal principal) {
    var userId = UserId.of(principal.getName());
    if (!roomService.isMember(RoomId.of(roomId), userId))
        throw new ForbiddenException();
    // ...
}
```

→ Principal 이 곧 user (JWT 검증 결과).

---

## 5. 함정

1. **CONNECT 후 검증 안 함** → anonymous user 가 모든 room SUBSCRIBE.
2. **JWT in URL query** → log / proxy 노출.
3. **만료 JWT 가 ESC** (인터셉터 무시) → middleware order.
4. **SUBSCRIBE 시 room 멤버 검증 X** → 가입 안 한 room 의 메시지 receive.
5. **Origin 검증 X** → CSRF.

---

## 6. 관련

- [[security|↑ hub]]
- [[../implementation/connect-auth-impl]]
- [[csrf-websocket]]
- [[../../signup/security/security|↗ signup JWT]]
