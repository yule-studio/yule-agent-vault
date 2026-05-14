---
title: "Multi-device sync — Redis Pub/Sub fan-out ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:10:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, multi-device]
---

# Multi-device sync — Redis Pub/Sub fan-out ★

**[[design-decisions|↑ hub]]**

> 카톡: A 가 폰에서 보낸 메시지가 태블릿 + 웹 에서도 즉시 보임 + 폰에서 읽음 → 태블릿도 읽음.

---

## 1. 본 vault — Spring `convertAndSendToUser` + Redis Pub/Sub

```
A's devices:
  Phone   → WebSocket session #1 (node A)
  Tablet  → WebSocket session #2 (node B)
  Web     → WebSocket session #3 (node A)

A 가 phone 으로 send:
  1. node A 가 message DB INSERT
  2. node A 가 Redis publish "user:A" + message
  3. node A 가 self-send /user/queue/sync → phone + web (같은 노드)
  4. node B 가 subscribe 받음 → /user/queue/sync → tablet
```

→ Spring 의 `SimpUserRegistry` + `convertAndSendToUser` + Redis backplane.

---

## 2. 왜 Redis Pub/Sub (sticky session 만 X)

### 2.1 단일 노드 (sticky)

- A 의 모든 device 가 같은 노드 → Spring `SimpUserRegistry` 가 자동 fan-out.
- 단점: 노드 1개의 부하 / scale-out X.

### 2.2 multi-node + sticky

- A 의 device 들이 다 같은 노드 — load 균형 X.
- 노드 down 시 모든 A 의 device 끊김.

### 2.3 multi-node + Redis Pub/Sub ★

- A 의 device 들이 다른 노드 가능 — 균형 OK.
- Redis publish → 모든 노드 subscribe → 해당 user 의 session 에 send.
- 본 vault 선택.

---

## 3. 흐름 (코드 패턴)

```java
@Service
@RequiredArgsConstructor
public class MultiDeviceSyncService {

    private final SimpMessagingTemplate messaging;
    private final StringRedisTemplate redis;
    private final ObjectMapper mapper;

    public void publishToUser(UserId user, String type, Object payload) {
        var envelope = Map.of("type", type, "payload", payload);

        // 1. 같은 노드의 session
        messaging.convertAndSendToUser(user.value(), "/queue/sync", envelope);

        // 2. 다른 노드 의 session (Redis)
        redis.convertAndSend("user-sync:" + user.value(), mapper.writeValueAsString(envelope));
    }
}

@Component
@RequiredArgsConstructor
public class CrossNodeSyncSubscriber implements MessageListener {

    private final SimpMessagingTemplate messaging;
    private final ObjectMapper mapper;

    public void onMessage(Message msg, byte[] pattern) {
        var channel = new String(msg.getChannel());   // "user-sync:abc"
        var userId = channel.split(":")[1];
        var envelope = mapper.readValue(msg.getBody(), Map.class);

        messaging.convertAndSendToUser(userId, "/queue/sync", envelope);
    }
}

@Configuration
public class RedisPubSubConfig {
    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory cf, CrossNodeSyncSubscriber sub) {
        var c = new RedisMessageListenerContainer();
        c.setConnectionFactory(cf);
        c.addMessageListener(sub, new PatternTopic("user-sync:*"));
        return c;
    }
}
```

자세히: [[../implementation/multi-device-sync-impl]].

---

## 4. Sync event types

| Type | trigger | 동기 |
| --- | --- | --- |
| `MESSAGE_SENT` | 본인이 보낸 메시지 | 다른 device 표시 |
| `READ_UPDATED` | 읽음 표시 | 다른 device 의 "안 읽은 수" 갱신 |
| `ROOM_JOINED` | 새 room 가입 | 다른 device 의 room 목록 |
| `ROOM_LEFT` | 퇴장 | room 목록 제거 |
| `PROFILE_UPDATED` | 프로필 변경 | 다른 device 갱신 |

---

## 5. 함정

1. **Redis 1 노드 만** → SPOF.
   → Redis cluster / Sentinel.
2. **Pub/Sub 의 message 손실** (subscriber 가 receive 도중 fail).
   → 중요 sync 는 DB 로도 (read 시 catchup).
3. **같은 노드 + Redis 둘 다 send** → 중복.
   → channel 가 노드 별 unique pattern 사용.
4. **메시지 size 큼** (첨부 포함) → Redis cluster overhead.
   → 메시지 ID 만 Pub/Sub, 나머지 DB 에서 fetch.
5. **subscriber thread 부족** → backpressure.

---

## 6. 다른 컨텍스트

- 카톡 / 라인: 자체 message broker + 별도 device session tracking.
- WhatsApp: phone 이 primary, 다른 device 는 web (sync via phone).
- Slack: WebSocket + RTM events.
- Discord: gateway protocol (자체 binary).

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[scale-strategy]]
- [[../implementation/multi-device-sync-impl]]
- [[../pitfalls/multi-device-pitfalls]]
