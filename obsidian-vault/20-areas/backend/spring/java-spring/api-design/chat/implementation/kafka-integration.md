---
title: "Kafka 통합 (F10+)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:08:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, kafka, advanced]
---

# Kafka 통합 (F10+)

**[[implementation|↑ hub]]**

> 설계: [[../design-decisions/kafka-event-driven]].

---

## 1. Producer (메시지 send → Kafka)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onMessageSent(MessageSent ev) {
    kafka.send("chat.message.v1",
        ev.roomId().value(),       // partition key
        mapper.writeValueAsString(ev));
}
```

---

## 2. Consumer (broadcast)

```java
@KafkaListener(topics = "chat.message.v1", groupId = "chat-broadcast")
public void onMessage(@Payload String json, Acknowledgment ack) {
    var ev = mapper.readValue(json, MessageSent.class);

    if (consumed.exists(ev.eventId())) {
        ack.acknowledge();
        return;
    }
    consumed.save(ev.eventId());

    ws.convertAndSend("/topic/room/" + ev.roomId().value(), ev.toDto());
    ack.acknowledge();
}
```

---

## 3. 다른 consumer

- `chat-notification-consumer` — offline push
- `chat-analytics-consumer` — 통계
- `chat-moderation-consumer` — AI 욕설 / 스팸 검출

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/kafka-event-driven]]
- [[../../product/implementation/kafka-integration|↗ product kafka]]
- [[../../notification/implementation/kafka-integration|↗ notification kafka]]
