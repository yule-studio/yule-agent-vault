---
title: "Multi-device sync 구현 ★ (Redis Pub/Sub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:54:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, multi-device]
---

# Multi-device sync 구현 ★ (Redis Pub/Sub)

**[[implementation|↑ hub]]**

> 설계: [[../design-decisions/multi-device-sync]].

---

## 1. Publisher (in-process + Redis)

```java
@Component
@RequiredArgsConstructor
public class RedisMessagePublisher implements MessagePublisher {

    private final StringRedisTemplate redis;
    private final SimpMessagingTemplate ws;
    private final ObjectMapper mapper;

    public void publishToRoom(RoomId room, Object payload) {
        var json = mapper.writeValueAsString(payload);
        redis.convertAndSend("room:" + room.value(), json);
    }

    public void publishToUser(UserId user, Object payload) {
        // 1) 같은 노드 의 user session
        ws.convertAndSendToUser(user.value(), "/queue/sync", payload);
        // 2) 다른 노드 의 user session
        var json = mapper.writeValueAsString(Map.of("user", user.value(), "payload", payload));
        redis.convertAndSend("user-sync", json);
    }
}
```

---

## 2. Subscriber (cross-node)

```java
@Component
@RequiredArgsConstructor
public class ChatMessageSubscriber implements MessageListener {

    private final SimpMessagingTemplate ws;
    private final ObjectMapper mapper;

    public void onMessage(Message msg, byte[] pattern) {
        var channel = new String(msg.getChannel());
        var body = new String(msg.getBody());

        if (channel.startsWith("room:")) {
            var roomId = channel.substring("room:".length());
            ws.convertAndSend("/topic/room/" + roomId, mapper.readValue(body, Object.class));
            return;
        }
        if (channel.equals("user-sync")) {
            var envelope = mapper.readValue(body, Map.class);
            var user = (String) envelope.get("user");
            ws.convertAndSendToUser(user, "/queue/sync", envelope.get("payload"));
        }
    }
}
```

---

## 3. Config

```java
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory cf, ChatMessageSubscriber sub) {
        var c = new RedisMessageListenerContainer();
        c.setConnectionFactory(cf);
        c.addMessageListener(sub, new PatternTopic("room:*"));
        c.addMessageListener(sub, new ChannelTopic("user-sync"));
        return c;
    }
}
```

---

## 4. 함정

- 같은 노드 + Redis 둘 다 broadcast → 중복.
   → 노드 마다 자신의 publish 는 self skip (메시지 origin 표시).
- Pub/Sub 손실 (subscriber 못 받음) → DB GET 으로 catchup.
- subscriber thread 부족 → backpressure.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/multi-device-sync]]
- [[../design-decisions/scale-strategy]]
- [[message-send-impl]]
