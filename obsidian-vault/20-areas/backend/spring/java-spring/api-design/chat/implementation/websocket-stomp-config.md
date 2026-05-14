---
title: "WebSocket STOMP config ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:42:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, websocket, stomp]
---

# WebSocket STOMP config ★

**[[implementation|↑ hub]]**

---

## 1. F0~F4 — SimpleBroker

```java
@Configuration
@EnableWebSocketMessageBroker
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final StompAuthInterceptor stompAuthInterceptor;

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins(
                "https://example.com",
                "https://app.example.com")
            .withSockJS();   // legacy fallback
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // F0~F4: SimpleBroker
        config.enableSimpleBroker("/topic", "/queue")
            .setHeartbeatValue(new long[] { 10_000, 10_000 })
            .setTaskScheduler(taskScheduler());

        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(stompAuthInterceptor);
    }

    @Bean
    public TaskScheduler taskScheduler() {
        var ts = new ThreadPoolTaskScheduler();
        ts.setPoolSize(4);
        ts.setThreadNamePrefix("ws-heartbeat-");
        return ts;
    }
}
```

---

## 2. F5+ — RabbitMQ STOMP relay

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")
        .setRelayHost("rabbit")
        .setRelayPort(61613)
        .setClientLogin(rabbitProps.user())
        .setClientPasscode(rabbitProps.pass())
        .setSystemLogin(rabbitProps.systemUser())
        .setSystemPasscode(rabbitProps.systemPass())
        .setVirtualHost("/")
        .setSystemHeartbeatSendInterval(10_000)
        .setSystemHeartbeatReceiveInterval(10_000);

    config.setApplicationDestinationPrefixes("/app");
    config.setUserDestinationPrefix("/user");
}
```

자세히: [[../design-decisions/scale-strategy]].

---

## 3. 본 vault config 정리

| 항목 | 값 |
| --- | --- |
| Endpoint | `/ws` |
| SockJS fallback | enabled (legacy browser) |
| User prefix | `/user` |
| App prefix | `/app` |
| Broker prefix | `/topic`, `/queue` |
| Heartbeat | 10s/10s |
| Allowed origins | production domain only |

---

## 4. 함정

1. **SetAllowedOrigins("*")** → CSRF.
2. **interceptor 순서** — auth 가 가장 먼저.
3. **TaskScheduler 없음** → heartbeat 동작 X.
4. **`/topic` 만 — `/queue` 빼먹음** → user-destination 동작 X.

---

## 관련

- [[implementation|↑ hub]]
- [[connect-auth-impl]]
- [[../security/websocket-auth]]
- [[../design-decisions/stomp-vs-raw]]
- [[../../websocket-stomp|↗ websocket-stomp recipe]]
