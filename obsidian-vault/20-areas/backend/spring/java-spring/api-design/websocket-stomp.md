---
title: "WebSocket + STOMP — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - websocket
  - stomp
  - realtime
---

# WebSocket + STOMP — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | STOMP broker / JWT 인증 / subscription 권한 |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]]. 응용 — [[chat-realtime]].

---

## 1. 무엇을 만드는가

**WebSocket** = TCP 위 양방향 통신. **STOMP** = WebSocket 위의 메시지 프로토콜 (subscribe / send / unsubscribe).

```
클라 → WebSocket 핸드셰이크 (CONNECT 프레임 + Authorization 헤더)
   ↓
서버 → JWT 검증 → Principal 결정
   ↓
클라 → SUBSCRIBE /topic/chat/{roomId}     (개인 — /user/queue/notifications)
   ↓
서버 → SimpMessagingTemplate.convertAndSendToUser(...) / convertAndSend(...)
```

### 1.1 endpoint

```
ws  /api/v1/ws                      (개발)
wss /api/v1/ws                      (운영)

브로드캐스트:  /topic/{...}          (다 같이 받음)
개인 큐:      /user/queue/{...}      (특정 user)
```

---

## 2. 의존성

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-websocket")
// 외부 broker (RabbitMQ) 사용 시:
// implementation("org.springframework.boot:spring-boot-starter-amqp")
```

---

## 3. WebSocket 설정

```java
// common/config/WebSocketConfig.java
@Configuration
@EnableWebSocketMessageBroker
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final StompAuthChannelInterceptor authInterceptor;

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/api/v1/ws")
            .setAllowedOriginPatterns("https://shop.example.com", "https://admin.example.com",
                                      "http://localhost:3000")
            // SockJS fallback (옛 브라우저 / corp 방화벽)
            .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // 내장 simple broker — 단일 인스턴스 OK
        registry.enableSimpleBroker("/topic", "/queue")
            .setHeartbeatValue(new long[]{ 10_000, 10_000 })           // server, client
            .setTaskScheduler(heartbeatScheduler());

        // 다중 인스턴스 + 외부 broker — RabbitMQ STOMP
        // registry.enableStompBrokerRelay("/topic", "/queue")
        //     .setRelayHost(rabbitHost).setRelayPort(61613)
        //     .setClientLogin(user).setClientPasscode(pwd)
        //     .setUserDestinationBroadcast("/topic/unresolved.user.dest");

        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(authInterceptor);
    }

    @Bean
    public TaskScheduler heartbeatScheduler() {
        var s = new ThreadPoolTaskScheduler();
        s.setPoolSize(2);
        s.setThreadNamePrefix("ws-heartbeat-");
        s.initialize();
        return s;
    }
}
```

### 3.1 내장 broker vs 외부 broker

| | 내장 (`enableSimpleBroker`) | 외부 (RabbitMQ STOMP) |
| --- | --- | --- |
| 인스턴스 수 | 1 (또는 sticky session) | N |
| 메시지 지속성 | 메모리 | 가능 (queue 영속) |
| 운영 | 단순 | broker 운영 부담 |
| 권장 | < 1만 동접 | 분산 / 영속 필요 |

→ 시작은 내장 + 단일 인스턴스. 성장 시 RabbitMQ relay.

---

## 4. STOMP CONNECT 시 JWT 인증

```java
// common/config/websocket/StompAuthChannelInterceptor.java
@Component
@RequiredArgsConstructor
@Slf4j
public class StompAuthChannelInterceptor implements ChannelInterceptor {

    private final JwtTokenProvider jwt;
    private final UserQueryService users;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        var accessor = StompHeaderAccessor.wrap(message);

        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            // Authorization: Bearer <jwt> 헤더로 받음
            var auth = accessor.getFirstNativeHeader("Authorization");
            if (auth == null || !auth.startsWith("Bearer ")) {
                throw new MessageDeliveryException("missing token");
            }
            var token = auth.substring(7);
            if (!jwt.validateToken(token) || !jwt.isAccess(token)) {
                throw new MessageDeliveryException("invalid token");
            }
            var email = jwt.getEmail(token);
            var user = users.findActiveByEmail(email);
            if (user == null) throw new MessageDeliveryException("user not found");

            var authentication = new UsernamePasswordAuthenticationToken(
                user, null,
                List.of(new SimpleGrantedAuthority(user.getRole().asAuthority()))
            );
            accessor.setUser(authentication);
            log.info("WS CONNECT user={}", user.getId());
        }

        if (StompCommand.SUBSCRIBE.equals(accessor.getCommand())) {
            // 구독 권한 검증 — §5
            checkSubscriptionPermission(accessor);
        }
        return message;
    }

    private void checkSubscriptionPermission(StompHeaderAccessor accessor) {
        var destination = accessor.getDestination();
        var principal = accessor.getUser();
        if (destination == null || principal == null) {
            throw new MessageDeliveryException("subscription requires auth");
        }
        // /user/queue/* — Spring 이 자동으로 user-scope 만 매칭
        // /topic/chat/{roomId} — 채팅방 멤버 검증 필요 (chat 레시피)
        if (destination.startsWith("/topic/chat/")) {
            var roomId = destination.substring("/topic/chat/".length());
            // chatMembership.canSubscribe(user, roomId) — 별도 service
        }
    }
}
```

---

## 5. 권한 — destination 별 검증

```java
@Component
@RequiredArgsConstructor
public class StompSubscriptionGuard {

    private final ChatRoomMembershipService chatMembers;

    public boolean canSubscribe(User user, String destination) {
        if (destination.startsWith("/topic/admin/")) {
            return user.getRole() == Role.ADMIN || user.getRole() == Role.MASTER;
        }
        if (destination.startsWith("/topic/chat/")) {
            var roomId = destination.substring("/topic/chat/".length());
            return chatMembers.isMember(user.getId(), roomId);
        }
        if (destination.startsWith("/user/queue/")) {
            return true;                              // Spring 이 user 매칭 자동
        }
        return false;                                  // default deny
    }
}
```

---

## 6. Send / Subscribe 패턴

### 6.1 broadcast (전체 / 채팅방)

```java
// 도메인 이벤트 listener
@Component
@RequiredArgsConstructor
public class ChatMessageBroadcaster {
    private final SimpMessagingTemplate ws;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(ChatMessageSent event) {
        ws.convertAndSend("/topic/chat/" + event.roomId(),
            new ChatMessageDto(event.messageId(), event.senderId(),
                               event.content(), event.sentAt()));
    }
}
```

### 6.2 user 개인 큐 — 알림

```java
ws.convertAndSendToUser(
    userId.value(),                                   // principal.getName()
    "/queue/notifications",
    new NotificationDto(...)
);
// 클라가 SUBSCRIBE /user/queue/notifications 하면 자기것만 받음
```

### 6.3 Controller — STOMP `@MessageMapping`

```java
@Controller
@RequiredArgsConstructor
public class ChatStompController {

    private final ChatService chatService;

    @MessageMapping("/chat/{roomId}/send")
    public void send(@DestinationVariable String roomId,
                     @Payload SendMessageRequest req,
                     Principal principal) {
        var senderId = ((User) ((Authentication) principal).getPrincipal()).getId();
        chatService.send(new ChatRoomId(roomId), senderId, req.content());
        // 도메인 이벤트가 broadcast 처리 (위 §6.1)
    }
}

public record SendMessageRequest(@NotBlank @Size(max = 4000) String content) {}
```

---

## 7. SecurityConfig — WebSocket endpoint permit

```java
// SecurityConfig.filterChain 에
.requestMatchers("/api/v1/ws/**").permitAll()    // 핸드셰이크는 HTTP — JWT 는 CONNECT 프레임에서
```

→ HTTP 핸드셰이크는 인증 검증 X (모든 origin 차단은 `setAllowedOriginPatterns`). 실제 인증은 STOMP CONNECT 프레임의 `StompAuthChannelInterceptor` 가.

---

## 8. heartbeat / 연결 끊김 감지

```java
registry.enableSimpleBroker("/topic", "/queue")
    .setHeartbeatValue(new long[]{ 10_000, 10_000 });
```

- 서버 → 클라: 10초마다 ping
- 클라 → 서버: 10초마다 ping
- 25초 무응답 = 연결 해제

클라이언트 (JS):
```javascript
const client = new StompJs.Client({
  brokerURL: 'wss://shop.example.com/api/v1/ws',
  connectHeaders: { Authorization: 'Bearer ' + accessToken },
  reconnectDelay: 5000,
  heartbeatIncoming: 10000,
  heartbeatOutgoing: 10000,
  onConnect: () => {
    client.subscribe('/user/queue/notifications', msg => {
      const noti = JSON.parse(msg.body);
      // ...
    });
    client.subscribe('/topic/chat/' + roomId, msg => { ... });
  }
});
client.activate();
```

---

## 9. 다중 인스턴스 — sticky session vs 외부 broker

### 9.1 sticky session (간단)

LB 가 같은 client = 같은 instance. `JSESSIONID` 또는 `X-Client-ID` 기반.

문제: 인스턴스 죽으면 그 사용자 연결 다 끊김. 메시지 broadcast 가 다른 인스턴스 user 에 안 감 (각 인스턴스의 SimpleBroker 는 자기 client 만 앎).

### 9.2 외부 broker (RabbitMQ STOMP)

```java
registry.enableStompBrokerRelay("/topic", "/queue")
    .setRelayHost(rabbitHost)
    .setRelayPort(61613)
    .setClientLogin(user)
    .setClientPasscode(passcode);
```

→ 모든 인스턴스가 RabbitMQ 를 통해 메시지 라우팅. 진정한 수평 확장.

### 9.3 Redis Pub/Sub (Spring Session WebSocket)

옵션 — `spring-session` + Redis pub/sub. RabbitMQ 운영 부담 없음. 단 message persistence X.

---

## 10. 함정 모음

### 함정 1 — HTTP 인증 의존
WebSocket 핸드셰이크는 HTTP 1.1 + Upgrade. 그러나 connection 후엔 HTTP 인증 무관. **STOMP CONNECT 프레임의 Authorization 헤더** 로 검증.

### 함정 2 — CORS `setAllowedOriginPatterns`
운영에선 명시. `*` X. SockJS 환경에선 `setAllowedOrigins` 와 차이 주의.

### 함정 3 — subscription 권한 미검증
누구나 `/topic/admin/...` 구독 가능 = 정보 노출. **`StompSubscriptionGuard`** 필수.

### 함정 4 — heartbeat 안 켬
ngrok / corp 방화벽이 TCP idle 끊음 → 클라가 모름. heartbeat 으로 자동 reconnect 트리거.

### 함정 5 — SimpleBroker 가 다중 인스턴스에 broadcast 안 됨
인스턴스 A 의 publish 가 B 의 구독자에 안 감. **외부 broker** 필요.

### 함정 6 — 메시지 무손실 기대
WebSocket = at-most-once. 중요 메시지는 **DB persistence + 폴링 백업**.

### 함정 7 — backpressure 없음
broker 가 클라보다 빠르게 publish 하면 메모리 폭발. 클라 read 지연 시 disconnect 처리.

### 함정 8 — `SimpMessagingTemplate.convertAndSendToUser` 의 principal name
Spring 의 user 식별 = `Principal.getName()`. 일관성 (UserId 사용 권장).

### 함정 9 — 도메인 이벤트 listener 가 트랜잭션 안에서 broadcast
DB commit 전에 메시지 발송 → rollback 시 잘못된 알림. **`@TransactionalEventListener(AFTER_COMMIT)`**.

### 함정 10 — WebSocket connection limit
서버 / Tomcat default 가 작음. `server.tomcat.max-connections` + OS file descriptor 한계.

---

## 11. 운영 체크리스트

- [ ] `wss://` 강제 (TLS)
- [ ] STOMP CONNECT 시 JWT 검증
- [ ] subscription 권한 검증 (destination 별)
- [ ] heartbeat 10s
- [ ] 다중 인스턴스 = 외부 broker (RabbitMQ STOMP) 또는 sticky session
- [ ] 동접 수 모니터 (`server.tomcat.max-connections`)
- [ ] disconnect 이유 logging (인증 실패 / heartbeat timeout / 정상)
- [ ] 도메인 이벤트로 broadcast trigger (AFTER_COMMIT)
- [ ] CORS origin 명시
- [ ] CSP 가 wss 도메인 허용

---

## 12. 관련

- [[chat-realtime]] — 본 레시피 응용
- 알림 (notification) — `convertAndSendToUser` 활용
- [[../common/security-config]] — STOMP CONNECT 인증
- [[../pitfalls/resource-leaks]] (예정) — connection limit
- [[api-design|↑ api-design hub]]
