---
title: "Kafka 통합 (F8+)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:04:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, kafka, advanced]
---

# Kafka 통합 (F8+)

**[[implementation|↑ hub]]**

> 자세한 설계: [[../design-decisions/kafka-event-driven]].

---

## 1. Producer (CDC + Debezium 또는 outbox-relay)

```java
@Component
@RequiredArgsConstructor
public class OutboxToKafkaRelay {

    private final NotificationOutboxRepository repo;
    private final KafkaTemplate<String, String> kafka;
    private final ObjectMapper mapper;

    @Scheduled(fixedDelay = 500)
    @SchedulerLock(name = "notifKafkaRelay")
    public void relay() {
        var pending = repo.findPendingForRelay(100);
        for (var row : pending) {
            kafka.send("notification.events.v1",
                row.userId().value(),       // partition key
                mapper.writeValueAsString(row));
        }
        repo.markRelayed(pending.stream().map(Notification::id).toList());
    }
}
```

---

## 2. Consumer (채널별)

```java
@Component
@RequiredArgsConstructor
public class FcmConsumer {

    private final ChannelRouter router;
    private final ConsumedEventRepository consumed;
    private final ObjectMapper mapper;

    @KafkaListener(
        topics = "notification.events.v1",
        groupId = "fcm-consumer",
        concurrency = "5")
    @RetryableTopic(
        attempts = "5",
        backoff = @Backoff(delay = 1000, multiplier = 2.0),
        dltTopicSuffix = ".dlq")
    public void onEvent(@Payload String json, Acknowledgment ack) {
        var event = mapper.readValue(json, NotificationEvent.class);

        // dedup
        if (consumed.exists(event.id())) { ack.acknowledge(); return; }
        consumed.save(event.id(), Instant.now());

        // FCM channel 만 처리
        if (event.channels().contains(ChannelType.FCM)) {
            router.routeFcm(event);
        }
        ack.acknowledge();
    }
}
```

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/kafka-event-driven]]
- [[../../product/implementation/kafka-integration|↗ product kafka]]
