---
title: "Kafka 통합 구현 ★ — Outbox + Producer + Consumer"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - kafka
  - advanced
---

# Kafka 통합 구현 ★ — Outbox + Producer + Consumer (F10+)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

> F10+ 고도화. design 은 [[../design-decisions/kafka-event-driven]] 참고.

---

## 1. Outbox 테이블

```sql
-- V40__create_event_outbox.sql
CREATE TABLE event_outbox (
    id           CHAR(26) PRIMARY KEY,
    event_id     CHAR(26) NOT NULL UNIQUE,
    topic        VARCHAR(100) NOT NULL,
    partition_key VARCHAR(100) NOT NULL,
    payload      TEXT NOT NULL,                  -- JSON
    published    BOOLEAN NOT NULL DEFAULT FALSE,
    published_at TIMESTAMPTZ,
    attempts     INTEGER NOT NULL DEFAULT 0,
    last_error   TEXT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_outbox_unpublished ON event_outbox (created_at)
    WHERE published = FALSE;
```

---

## 2. Producer (outbox 워커)

```java
@Component
@RequiredArgsConstructor
public class OutboxPublishWorker {

    private final EventOutboxRepository outbox;
    private final KafkaTemplate<String, String> kafka;
    private final Clock clock;

    @Scheduled(fixedDelay = 100)
    @SchedulerLock(name = "outboxWorker", lockAtMostFor = "1m")
    public void publish() {
        var rows = outbox.findUnpublishedForUpdate(100);     // SKIP LOCKED
        for (var row : rows) sendOne(row);
    }

    @Transactional
    public void sendOne(EventOutbox row) {
        try {
            kafka.send(row.topic(), row.partitionKey(), row.payload())
                .get(5, TimeUnit.SECONDS);
            row.markPublished(clock.now());
            outbox.save(row);
        } catch (Exception e) {
            row.recordFailure(e.getMessage());
            outbox.save(row);
        }
    }
}
```

---

## 3. Listener → outbox (예: 결제 승인)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentApproved(PaymentApproved ev) {
    outbox.save(EventOutbox.of(
        EventId.next(),
        "product.payment.events.v1",
        ev.orderId().value(),                  // partition key (순서 보장)
        mapper.writeValueAsString(toEnvelope(ev))));
}

private EventEnvelope toEnvelope(DomainEvent ev) {
    return new EventEnvelope(
        ev.eventId(),
        ev.getClass().getSimpleName(),
        "v1",
        ev.occurredAt(),
        ev.aggregateId(),
        ev.aggregateType(),
        ev);
}
```

---

## 4. Consumer (디지털 워커)

```java
@Component
@RequiredArgsConstructor
public class DigitalDeliveryConsumer {

    private final DigitalDeliveryService service;
    private final ConsumedEventRepository consumed;

    @KafkaListener(
        topics = "product.payment.events.v1",
        groupId = "digital-delivery-consumer",
        concurrency = "3")
    @RetryableTopic(
        attempts = "5",
        backoff = @Backoff(delay = 1000, multiplier = 2.0),
        dltTopicSuffix = ".dlq")
    public void onPaymentApproved(
            @Payload String rawPayload,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        var envelope = mapper.readValue(rawPayload, EventEnvelope.class);

        // dedup
        if (consumed.exists(envelope.eventId())) {
            ack.acknowledge();
            return;
        }
        consumed.save(new ConsumedEvent(envelope.eventId(), partition, offset, Instant.now()));

        if (!"PaymentApproved".equals(envelope.eventType())) {
            ack.acknowledge();
            return;
        }

        var ev = mapper.convertValue(envelope.payload(), PaymentApproved.class);
        service.startForOrder(ev.orderId(), ev.buyerId());

        ack.acknowledge();
    }

    @DltHandler
    public void onDlq(String rawPayload,
                       @Header(KafkaHeaders.EXCEPTION_MESSAGE) String err) {
        log.error("dlq: {} - {}", err, rawPayload);
        dlqRepo.save(new DlqRow(rawPayload, err, Instant.now()));
        slack.alert("payment dlq", err);
    }
}
```

---

## 5. config

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory(KafkaProperties props) {
        return new DefaultKafkaProducerFactory<>(Map.of(
            "bootstrap.servers", props.getBootstrapServers(),
            "key.serializer", StringSerializer.class,
            "value.serializer", StringSerializer.class,
            "acks", "all",                      // 안전성 우선
            "enable.idempotence", "true",
            "compression.type", "snappy",
            "linger.ms", 10,
            "max.in.flight.requests.per.connection", 5
        ));
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> cf) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(cf);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

---

## 6. Docker / EC2 배포 (참고)

`docker-compose.yml` 의 broker / kafka-ui 셋업은 [[../design-decisions/kafka-event-driven#71]] 참고.

EC2 production:
- AWS MSK (관리형) 또는 self-hosted 3 노드.
- SASL/PLAIN + TLS.

---

## 7. 함정

### 함정 1 — outbox 없이 직접 send
DB commit 후 Kafka timeout → 이벤트 손실.

### 함정 2 — acks=1 / acks=0
broker 1개 fail 시 이벤트 손실.
→ acks=all + min.insync.replicas=2.

### 함정 3 — dedup 안 함
at-least-once 의 중복 처리.
→ event_id UNIQUE.

### 함정 4 — partition key 잘못
순서 X.
→ aggregateId.

### 함정 5 — Manual ack 아닌 auto
처리 도중 commit → 손실 / 중복.
→ MANUAL_IMMEDIATE.

---

## 8. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/kafka-event-driven]]
- [[payment-confirm-impl]]
- [[digital-delivery-impl]]
- [[../testing/integration-tests]]
