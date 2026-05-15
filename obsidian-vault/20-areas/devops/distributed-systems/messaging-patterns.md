---
title: "Messaging patterns — Kafka / SQS / outbox / DLQ"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:51:00+09:00
tags: [devops, distributed-systems, messaging, kafka]
---

# Messaging patterns — Kafka / SQS / outbox / DLQ

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 도구

| | type | 강점 |
| --- | --- | --- |
| **Kafka** | log-based | high throughput / ordering / replay |
| **RabbitMQ** | broker | rich routing |
| **SQS** | SaaS | simple / managed |
| **SNS** | pub/sub | fanout |
| **Kinesis** | AWS Kafka-like | AWS 통합 |
| **NATS** | lightweight | edge / low latency |
| **Pulsar** | log + queue | multi-tenancy |
| **Redis Streams** | log light | 빠름 / 작음 |
| **EventBridge** | event routing | SaaS |

→ throughput / ordering / replay = Kafka. simple = SQS.

---

## 2. delivery guarantee

```
at-most-once:    가능하면 send, 잃어도 OK    (UDP-like)
at-least-once:   적어도 1번 (중복 가능)        (★ 흔함)
exactly-once:    정확히 1번 (어려움, 비싸)
```

→ "exactly-once" 는 producer + broker + consumer 의 협력 필요.

---

## 3. Kafka 핵심

```
Topic = partition 들의 묶음
Partition = 순서 보장된 log
Offset = partition 안의 위치
Consumer Group = 같은 group 의 consumer 들이 partition 분담

producer → partition (key 의 hash 또는 round-robin)
consumer 들이 group → partition 별 1 consumer
```

```
Topic: orders (3 partitions)
  Partition 0: o1, o2, o3, ...
  Partition 1: o5, o7, ...
  Partition 2: o4, o6, o8, ...

Consumer Group A (3 consumer):
  consumer-0 → partition 0
  consumer-1 → partition 1
  consumer-2 → partition 2

→ partition 안 ordering 보장. partition 간 X.
```

---

## 4. Kafka producer 설정

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", JsonSerializer.class.getName());

// reliability
props.put("acks", "all");                    // all replica ack
props.put("retries", Integer.MAX_VALUE);
props.put("max.in.flight.requests.per.connection", 5);
props.put("enable.idempotence", true);        // ★ exactly-once producer

// performance
props.put("compression.type", "lz4");
props.put("batch.size", 16384);
props.put("linger.ms", 10);                   // batch 위해 wait
```

---

## 5. Kafka consumer 설정

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "order-processor");
props.put("auto.offset.reset", "earliest");   // 또는 latest

// reliability
props.put("enable.auto.commit", "false");      // ★ manual commit (safer)
props.put("isolation.level", "read_committed"); // exactly-once

// performance
props.put("max.poll.records", 500);
props.put("fetch.max.bytes", 50 * 1024 * 1024);
```

```java
KafkaConsumer<String, OrderEvent> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));

while (true) {
    ConsumerRecords<String, OrderEvent> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, OrderEvent> record : records) {
        try {
            process(record.value());
        } catch (Exception e) {
            // DLQ 또는 retry
            handleFailure(record, e);
        }
    }
    consumer.commitSync();   // 처리 성공 시 offset 저장
}
```

---

## 6. partitioning 전략

```
key 선택:
  user_id        — 같은 user 의 message 순서 보장
  order_id       — 같은 order 순서
  none (round-robin) — load balance, 순서 X

partition 수:
  - 너무 적음 = throughput limit
  - 너무 많음 = overhead (broker / consumer)
  - 일반 = consumer 수의 2-3 배

증가는 가능, 감소 불가.
```

→ partition 수가 max concurrency.

---

## 7. ordering

```
보장:
  - partition 안의 message 들 순서
  - same key → same partition

위배:
  - partition 간 (둘 다 합쳐 보면 mix)
  - retry 시 reorder (max.in.flight.requests > 1 + retries)
  
해결:
  - max.in.flight.requests=1 (단 throughput ↓)
  - idempotent producer + retries (★ Kafka 권장)
```

---

## 8. consumer lag (★ 핵심 metric)

```
lag = producer offset - consumer offset

증가 = consumer 가 따라잡지 못함:
  - 처리 느림
  - consumer 부족
  - error → block

해결:
  - scale-out (consumer 추가)
  - 처리 logic 최적화
  - partition 늘림 (max parallel ↑)
```

```promql
kafka_consumergroup_lag{group="order-processor"}
```

---

## 9. dead letter queue (DLQ) (★)

```
처리 fail message:
  무한 retry = 다른 message 도 block

DLQ pattern:
  1. message process
  2. fail 시 N retry (exponential backoff)
  3. 그래도 fail → DLQ topic 으로 publish
  4. main topic 의 offset commit (다음으로 진행)
  5. DLQ 는 별도 analyze / manual reprocess
```

```java
@KafkaListener(topics = "orders")
public void onOrder(OrderEvent event) {
    try {
        process(event);
    } catch (RetryableException e) {
        throw e;   // retry
    } catch (NonRetryableException e) {
        // DLQ
        kafkaTemplate.send("orders.DLQ", event);
        log.error("sent to DLQ", e);
    }
}
```

Spring Kafka 의 자동 DLQ:
```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2.0),
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "orders")
public void onOrder(OrderEvent event) {
    process(event);
}
```

→ 자동 retry + DLQ topic 생성.

---

## 10. outbox pattern (★ DB + Kafka 의 consistency)

```
문제: DB save + Kafka publish 의 atomic 보장
  - DB OK, Kafka fail → lost event
  - Kafka OK, DB fail → 잘못된 event

해결: outbox
  1. tx 안: INSERT INTO orders + INSERT INTO outbox
  2. 별도 process: outbox → Kafka publish + mark processed
  3. DB 의 ACID 활용

도구:
  - Debezium (CDC, Kafka Connect)
  - 또는 polling worker
```

```java
@Transactional
public Order create(OrderRequest req) {
    Order order = repo.save(new Order(req));
    
    outboxRepo.save(new OutboxEvent(
        UUID.randomUUID(),
        "OrderCreated",
        order.getId(),
        JsonUtil.toJson(order)
    ));
    
    return order;
}

// 별도 worker
@Scheduled(fixedDelay = 1000)
public void publishOutbox() {
    List<OutboxEvent> events = outboxRepo.findUnpublished(100);
    for (OutboxEvent e : events) {
        kafkaTemplate.send(e.getTopic(), e.getPayload());
        e.markPublished();
    }
}
```

---

## 11. exactly-once (★)

```
Kafka exactly-once semantics (EOS):
  - producer.enable.idempotence=true (no duplicate)
  - producer transactions (atomic write to N partition)
  - consumer.isolation.level=read_committed (안 commit msg 안 봄)

→ Kafka 안에서만 EOS.
application + 외부 system 까지 = 의 idempotency 도 필요.
```

---

## 12. fanout (★ multiple consumer)

```
한 message → 여러 consumer 가 각자 처리:

Kafka 의 multiple consumer group:
  Group A: order-billing
  Group B: order-notification  
  Group C: order-analytics
  
→ 각 group 독립적 offset.
→ Group A 만 lag 있어도 Group B 영향 X.

SNS + SQS:
  SNS topic → multiple SQS queue 로 fanout
```

---

## 13. message schema (★)

```
producer 와 consumer 의 호환성:

방법:
  - JSON (★ 흔함, schema 관리 약함)
  - Avro + Schema Registry (★ 권장)
  - Protobuf
  - Thrift

Confluent Schema Registry:
  - producer 가 schema 등록
  - consumer 가 schema 검증
  - backward / forward compatibility 강제
```

---

## 14. 운영 (★)

```
broker:
  - 3+ broker (RF=3)
  - SSD (큰 IO)
  - 충분한 heap (Kafka 는 OS page cache 의존)
  - tiered storage (오래된 segment → S3, 비용 절감)

monitoring:
  - consumer lag
  - under-replicated partitions
  - ISR (in-sync replicas)
  - throughput
  - latency

backup:
  - MirrorMaker 2 (다른 cluster 복제)
  - tiered storage (S3 archive)
```

---

## 15. 함정

1. **partition 너무 적음** — consumer scale 제한.
2. **partition 너무 많음** — broker overhead.
3. **key 잘못** — hot partition (한 partition 만).
4. **auto.offset.reset = latest** — 누락.
5. **manual commit 안 함** — 처리 전 commit → lost.
6. **DLQ 없이 무한 retry** — block / 폭주.
7. **outbox 없이 DB + Kafka** — atomicity 깨짐.
8. **schema 변경 호환성 없음** — consumer break.

---

## 16. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[saga-pattern]]
- [[idempotency]]
- [[event-sourcing-cqrs]]
