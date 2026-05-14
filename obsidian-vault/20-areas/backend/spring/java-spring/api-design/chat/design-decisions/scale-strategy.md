---
title: "Scale strategy — Redis Pub/Sub + sticky session + RabbitMQ relay ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:26:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, scale]
---

# Scale strategy — Redis Pub/Sub + sticky session + RabbitMQ relay ★

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault 단계

| Phase | 구성 | 동시 connection |
| --- | --- | --- |
| F0~F4 | 단일 노드 + SimpleBroker | ~ 1만 |
| F5 | 다중 노드 + Redis Pub/Sub backplane | ~ 5만 |
| F6+ | RabbitMQ STOMP relay | ~ 50만 |
| F10+ | Kafka 분산 + multi region | 무한 |

---

## 2. 왜 Redis Pub/Sub (단순 sticky 아님)

### 2.1 sticky session 만

- nginx ip_hash / ALB sticky cookie.
- 사용자 의 모든 device 같은 노드 → Spring `SimpUserRegistry` fan-out.
- 문제: 노드 down → 그 노드의 모든 user 끊김.
- 문제: 노드 추가 → 균형 깨짐 (옛 user 그대로).

### 2.2 sticky + Redis Pub/Sub

- 각 노드가 user-sync:{userId} 채널 subscribe.
- 노드 간 메시지 fan-out — 다른 노드의 session 도 receive.
- 본 vault 선택.

### 2.3 RabbitMQ STOMP relay (F5+)

- broker 가 RabbitMQ → 무한 scale.
- STOMP 표준 — 코드 변경 없이 broker 만 교체.

---

## 3. Sticky session 구성

```nginx
# nginx
upstream chat_servers {
    ip_hash;
    server chat-1:8080;
    server chat-2:8080;
    server chat-3:8080;
}

# AWS ALB
target-group:
  stickiness: app_cookie (lb_cookie 도 가능)
  cookie-duration: 1 day
```

---

## 4. Redis Pub/Sub backplane

자세히: [[multi-device-sync#3]] 참고.

```java
@Bean
public RedisMessageListenerContainer container(
        RedisConnectionFactory cf, ChatMessageSubscriber sub) {
    var c = new RedisMessageListenerContainer();
    c.setConnectionFactory(cf);
    c.addMessageListener(sub, new PatternTopic("room:*"));
    c.addMessageListener(sub, new PatternTopic("user-sync:*"));
    return c;
}
```

---

## 5. RabbitMQ relay (F5+)

```yaml
spring:
  rabbitmq:
    host: rabbit
    port: 61613
    username: ${RABBIT_USER}
    password: ${RABBIT_PASS}
```

```java
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")
        .setRelayHost("rabbit")
        .setRelayPort(61613)
        .setClientLogin(rabbitUser)
        .setClientPasscode(rabbitPass);
    config.setApplicationDestinationPrefixes("/app");
}
```

---

## 6. 노드 별 connection 한도

| 노드 | OS file descriptor | JVM heap | Spring config |
| --- | --- | --- | --- |
| 1만 connection | `ulimit -n 65536` | 4GB+ | `server.tomcat.threads.max=200` |
| 5만 connection | `ulimit -n 200000` | 16GB | Reactor / WebFlux + Netty |
| 50만+ | k8s pod 다중 + 클러스터 | — | — |

---

## 7. 함정

1. **sticky session 없음** → 매번 다른 노드 → session 끊김.
2. **Redis 1 노드** → SPOF.
3. **RabbitMQ relay 의존 후 fallback X** → Rabbit down 시 모든 chat 중단.
4. **노드 down 시 graceful disconnect** 없음 → 사용자가 reconnect 까지 메시지 손실.
5. **file descriptor 한도** 기본 1024 → 1만 connection 못 받음.
6. **GC pause** 큰 heap → 메시지 latency spike.

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[multi-device-sync]]
- [[kafka-event-driven]]
- [[../operations/scaling]]
- [[../implementation/websocket-stomp-config]]
