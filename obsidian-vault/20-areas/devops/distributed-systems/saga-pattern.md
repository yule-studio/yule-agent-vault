---
title: "Saga pattern — distributed transaction 대안"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:41:00+09:00
tags: [devops, distributed-systems, saga, transaction]
---

# Saga pattern — distributed transaction 대안

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 문제

```
MSA 의 transaction:
  Order Service:     create order
  Payment Service:   charge card
  Inventory Service: deduct stock
  Notification:      send email

→ 4 service 의 한 트랜잭션?
→ 2PC (distributed transaction) 가능? 
→ 비싸 + 느림 + complex
```

---

## 2. 2PC (2-Phase Commit) 한계

```
phase 1: prepare
  coordinator: 모두에게 "ready?"
  service 들: "yes" / "no"

phase 2: commit / abort
  모두 yes → commit
  하나라도 no → abort

문제:
  - coordinator SPOF
  - service 가 prepare 후 죽으면 lock
  - 느림 (모두 wait)
  - 외부 service (Stripe 등) 지원 X
```

→ MSA 에서 2PC 거의 안 씀.

---

## 3. Saga (★ 대안)

```
큰 transaction → 작은 local transaction 들 + compensation

성공 path:
  Step 1: create order            (commit)
  Step 2: charge card              (commit)
  Step 3: deduct stock             (commit)
  Step 4: send email               (commit)

step 3 실패:
  Step 3 fail
  → compensation step 2 (refund)
  → compensation step 1 (cancel order)
  
→ ACID 의 "atomic" 포기, eventual consistency.
```

---

## 4. 2 종류

### A. choreography (★ 단순, 분산)

```
각 service 가 이벤트 발행 + 들음:

[Order]   → OrderCreated  → [Payment]
[Payment] → PaymentDone   → [Inventory]
[Inventory] → StockDeducted → [Notification]

fail 시:
  [Inventory] → StockFailed → [Payment] (refund)
  [Payment]   → PaymentRefunded → [Order] (cancel)
```

장점:
- service 결합 ↓
- 단순 (한 코디네이터 없음)

단점:
- flow 추적 어려움
- cyclic dependency 가능
- monitoring 어려움

### B. orchestration (중앙 코디네이터)

```
[Saga Orchestrator]
  ↓
  ├─ command: createOrder       ← Order Service
  ├─ command: chargePayment     ← Payment
  ├─ command: deductStock       ← Inventory
  └─ command: sendNotification  ← Notification

fail:
  Orchestrator 가 compensation 호출.
```

장점:
- flow 명시
- monitoring 쉬움
- complex flow 가능

단점:
- orchestrator SPOF / coupling
- 추가 service

→ **simple = choreography**, **complex = orchestration**.

---

## 5. orchestration 도구

| | 무엇 |
| --- | --- |
| **Camunda / Zeebe** | BPMN workflow |
| **Temporal** (★) | Go / Java, durable workflow |
| **AWS Step Functions** | serverless |
| **Netflix Conductor** | Java |
| **Apache Airflow** | data / batch |
| **Spring State Machine** | 작음 |

→ 모던 = **Temporal**. AWS native = Step Functions.

---

## 6. Temporal 예 (★)

```java
@WorkflowInterface
public interface OrderWorkflow {
    @WorkflowMethod
    String createOrder(OrderRequest req);
}

public class OrderWorkflowImpl implements OrderWorkflow {
    private final OrderActivities activities = Workflow.newActivityStub(...);
    
    @Override
    public String createOrder(OrderRequest req) {
        String orderId = activities.createOrder(req);
        
        try {
            activities.chargePayment(orderId);
            activities.deductStock(orderId);
            activities.sendNotification(orderId);
            return orderId;
        } catch (Exception e) {
            // compensation
            activities.cancelOrder(orderId);
            throw e;
        }
    }
}
```

→ Temporal 이 state / retry / timeout / persistence 자동.

---

## 7. choreography 예 (Kafka)

```java
@KafkaListener(topics = "OrderCreated")
public void onOrderCreated(OrderCreated event) {
    try {
        paymentService.charge(event.orderId, event.amount);
        kafkaTemplate.send("PaymentCompleted", new PaymentCompleted(event.orderId));
    } catch (Exception e) {
        kafkaTemplate.send("PaymentFailed", new PaymentFailed(event.orderId, e.getMessage()));
    }
}

@KafkaListener(topics = "PaymentFailed")
public void onPaymentFailed(PaymentFailed event) {
    orderService.cancel(event.orderId);
    kafkaTemplate.send("OrderCancelled", new OrderCancelled(event.orderId));
}
```

→ Kafka 의 ordering / durability 활용.

---

## 8. compensation 작성 원칙

```
1. idempotent (★)
   같은 compensation 여러 번 OK
   
2. retry-safe
   network fail → retry
   
3. order matter
   reverse order of forward steps
   (last in, first out)
   
4. backward recovery
   완료된 step 의 effect 되돌림
   
5. 일부는 unrollback
   (이메일 보냄 → 못 되돌림 → 사과 메시지)
```

```java
// idempotent compensation
public void cancelOrder(String orderId) {
    Order order = repo.findById(orderId);
    if (order.getStatus() == "CANCELLED") {
        return;   // 이미 처리. 더 안 함.
    }
    order.cancel();
    repo.save(order);
}
```

---

## 9. outbox pattern (★ Kafka + DB consistency)

```
문제:
  DB 에 order save + Kafka 에 event publish
  → 둘 사이 atomicity 없음 → 한쪽 실패 시 inconsistent

해결: outbox pattern
  1. tx 안에서:
     INSERT INTO orders (...)
     INSERT INTO outbox (event_type, payload)
  2. 별도 process (CDC or polling):
     outbox 의 record 읽어서 Kafka 에 publish + 표시
  
→ DB 의 ACID 활용. Kafka 의 at-least-once.

도구: Debezium (Kafka Connect)
```

---

## 10. inbox pattern (consumer side)

```
consumer 가 같은 event 2번 받음 (at-least-once) → 2번 처리?

해결:
  1. event ID + dedup table
  2. INSERT INTO processed_events (event_id) — unique constraint
  3. 이미 있음 → skip
  4. 새 → 처리
```

→ idempotency 의 표준 패턴.

---

## 11. state machine 으로 표현

```
Order state:
  CREATED → PAYMENT_PENDING → PAYMENT_DONE → STOCK_DEDUCTED → CONFIRMED
       ↓             ↓                ↓                ↓
   CANCELLED   PAYMENT_FAILED   STOCK_FAILED      ERROR
                  → REFUNDED       → REFUNDED
                  → CANCELLED      → CANCELLED
```

→ 명시적 state. unknown state 없어야.

---

## 12. saga timeout

```
service 가 응답 안 함 (network / process down):
  saga 는 영원히 wait?
  
해결:
  - timeout 설정 (각 step)
  - timeout 시 자동 compensate
  
Temporal:
  withTimeout(Duration.ofSeconds(30))
  
Kafka:
  scheduled event (timeout)
  → consumer 가 처리
```

---

## 13. observability

```
saga 추적 핵심:
  - saga ID (correlation ID)
  - 각 step 시작 / 종료 시간
  - 현재 state
  - compensation 발화

도구:
  - Temporal UI (workflow 시각화)
  - Jaeger / Tempo (distributed trace)
  - 자체 dashboard (state 분포)
```

---

## 14. 함정

1. **compensation 작성 안 함** — 실패 시 inconsistent.
2. **idempotency 부재** — 중복 처리.
3. **timeout 없음** — 영원 wait.
4. **complex saga** (10+ step) — 디버그 지옥.
5. **choreography + complex flow** — orchestrate 권장.
6. **outbox 없이 DB + Kafka** — race condition.
7. **state machine 모호** — unknown state.
8. **compensation 의 compensation 필요** — 디자인 fail.

---

## 15. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[idempotency]]
- [[messaging-patterns]]
- [[event-sourcing-cqrs]]
- [[../../60-recipes/spring/product/payment/payment|↗ payment saga]]
