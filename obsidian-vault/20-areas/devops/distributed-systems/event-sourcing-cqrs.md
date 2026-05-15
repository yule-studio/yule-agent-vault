---
title: "Event sourcing / CQRS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:53:00+09:00
tags: [devops, distributed-systems, event-sourcing, cqrs]
---

# Event sourcing / CQRS

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. event sourcing 정의

```
전통 CRUD:
  orders table: {id, status, amount}
  UPDATE orders SET status='SHIPPED'  ← 옛 state 사라짐

event sourcing:
  events: 
    [OrderCreated, PaymentReceived, OrderShipped]
  state = events 의 sum (replay)

→ 모든 변경이 event. event 가 source of truth.
```

---

## 2. 왜

```
1. audit trail
   "왜 이 state 가 됐나" → event 로 정확

2. time-travel
   "어제 14시 의 state" → events replay until 14:00

3. debug
   "이 user 에게 무슨 일이?" → event 시퀀스

4. 새 view / projection
   같은 events 로 다른 view (read model)

5. integration
   event 자체가 다른 service 의 input

→ regulator / 금융 / 게임 / collaborative app 에 적합.
```

---

## 3. 단점 / 트레이드오프

```
- 학습 곡선 큼
- replay 시간 (수백만 event 면 느림)
- snapshot 필요 (성능)
- schema evolution 어려움
- 쿼리 어려움 (state 가 단순 SELECT 안 됨)
- developer 의 mind shift

→ 모든 service 가 event sourcing X. 부분만 적용 흔함.
```

---

## 4. 구조

```
[Command]
   ↓
[Aggregate]
   ↓ event publish
[Event Store]
   ↓
[Projections (Read Model)]
   ↓
[Query]
```

---

## 5. event store

```
Append-only log:
  global_position | aggregate_id | event_type      | payload
  1               | order-123    | OrderCreated    | {...}
  2               | order-123    | PaymentReceived | {...}
  3               | order-456    | OrderCreated    | {...}
  4               | order-123    | OrderShipped    | {...}

→ aggregate 별 stream + global ordering.
```

도구:
- **EventStore** (DB) — 전용
- **PostgreSQL** — 간단한 table
- **Kafka** — partition = stream
- **AWS Kinesis** + DynamoDB

```sql
-- 간단한 Postgres event store
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(64),
    sequence_no INT,
    event_type VARCHAR(64),
    payload JSONB,
    occurred_at TIMESTAMP,
    UNIQUE (aggregate_id, sequence_no)
);

CREATE INDEX ON events (aggregate_id, sequence_no);
```

---

## 6. aggregate (DDD)

```java
public class Order {
    private OrderId id;
    private OrderStatus status;
    private Money total;
    private List<Event> uncommitted = new ArrayList<>();
    
    // command
    public void ship() {
        if (status != OrderStatus.PAID) {
            throw new InvalidStateException();
        }
        Event event = new OrderShipped(id, Instant.now());
        apply(event);
        uncommitted.add(event);
    }
    
    // event 적용 (state 변경)
    private void apply(Event event) {
        switch (event) {
            case OrderCreated e -> { 
                id = e.orderId(); 
                status = OrderStatus.CREATED; 
                total = e.total(); 
            }
            case OrderShipped e -> status = OrderStatus.SHIPPED;
            // ...
        }
    }
    
    // replay from events
    public static Order from(List<Event> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }
}
```

---

## 7. snapshot

```
state 100만 event 누적 → replay 느림.

snapshot:
  every N events:
    aggregate 의 현재 state 저장
  
load 시:
  최신 snapshot 로드 + 그 후 events 만 replay
```

```sql
CREATE TABLE snapshots (
    aggregate_id VARCHAR(64),
    sequence_no INT,
    state JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (aggregate_id)
);
```

→ "1000 event 마다 snapshot" 정도.

---

## 8. CQRS (Command Query Responsibility Segregation)

```
전통:
  같은 model 로 read + write
  → read 와 write 의 요구 차이 → 한쪽 sub-optimal

CQRS:
  Write model (Aggregate, normalize, consistency)
  Read model (denormalize, optimize for query)
  
event 로 둘 연결:
  Write → event publish → Read model update
```

```
Write:
  POST /orders  → Order aggregate (DB)
  → "OrderCreated" event publish

Read:
  GET /orders/123 → Read model (별도 table)
  Read model 은 event 받아 update
  
  GET /orders?user=alice&status=SHIPPED&...
  → optimized denormalized table
```

---

## 9. CQRS 의 read model 예

```sql
-- write model (normalize)
orders (id, user_id, status, ...)
order_items (id, order_id, product_id, ...)
products (id, name, ...)

-- read model (denormalize for search)
order_search (
    order_id, user_name, status, total_amount, 
    item_names ARRAY, item_count, created_at
)

-- 한 쿼리로 모든 정보. fast read.
```

```java
// event handler
@EventListener
public void onOrderCreated(OrderCreated event) {
    // read model update
    OrderSearch row = new OrderSearch(
        event.orderId(),
        userRepo.getName(event.userId()),
        "CREATED",
        event.total(),
        // ...
    );
    readModelRepo.save(row);
}
```

---

## 10. eventual consistency in CQRS

```
write OK → event publish → read model update
                            ↑
                            잠시 lag

사용자 영향:
  - "주문하고 list 에 안 보임" 1-2초
  - 거의 늘 OK
  - critical 면 read-your-writes (write 시 cache)
```

---

## 11. projection 다양화

```
같은 events → 여러 read model:

events ─→ OrderSearch          (검색용)
       ─→ UserDashboard         (user 의 주문 list)
       ─→ DailyRevenue          (집계 / report)
       ─→ FraudDetection        (ML feature)
       ─→ Notification          (이메일 보내기)

→ 새 view 추가 = 새 projection.
→ 옛 events replay → 새 view 부터 채움.
```

---

## 12. schema evolution (★ 어려움)

```
event 의 schema 변경:
  v1: {orderId, amount}
  v2: {orderId, amount, currency}    ← currency 추가

옛 v1 event 의 처리:
  - default value (USD)
  - upcaster (v1 → v2 변환)
  - polymorphic event handler

이름 변경:
  - 새 event type
  - 옛 type 호환 유지 (deprecation period)
```

```java
// upcaster
public OrderCreatedV2 upcast(OrderCreatedV1 v1) {
    return new OrderCreatedV2(v1.orderId, v1.amount, "USD");
}
```

---

## 13. 도구

| | 무엇 |
| --- | --- |
| **Axon Framework** (Java) | event sourcing + CQRS |
| **AxonServer** | event store |
| **EventStore (Kurrent)** | DB |
| **Marten** (.NET) | Postgres 기반 |
| **EventFlow** | C# |
| **Eventuate** | polyglot |
| **자체** | Postgres + Kafka |

---

## 14. event-driven architecture (★ 비슷한 개념)

```
event sourcing != event-driven architecture (EDA).

EDA:
  service 간 통신을 event 로 (sync API 대신).
  → loose coupling.
  → 각 service 의 내부 state 는 보통 CRUD.

ES:
  service 의 state 가 event.

→ EDA 가 더 일반적. ES 는 일부 service.
```

---

## 15. 함정

1. **모든 service 가 ES** — over-engineering.
2. **schema evolution 무시** — 옛 event 손해.
3. **replay 시간** — snapshot 필수.
4. **event 의 size 큼** — DB / Kafka 부담.
5. **PII in event** — GDPR (right to be forgotten 어려움).
6. **eventual consistency 의 영향 평가 안 함** — UX 문제.
7. **command + query 같은 model** — CQRS 의미 X.

---

## 16. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[messaging-patterns]]
- [[saga-pattern]]
- [[../../60-recipes/spring/architecture/architecture|↗ architecture]]
