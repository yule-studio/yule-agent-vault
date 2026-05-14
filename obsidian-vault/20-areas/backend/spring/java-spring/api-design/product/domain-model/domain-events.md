---
title: "도메인 이벤트 — 16 events"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:32:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - event
---

# 도메인 이벤트 — 16 events

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. 이벤트 목록

| Event | aggregate | 트리거 | 처리 (F0~F9) | 처리 (F10+ Kafka) |
| --- | --- | --- | --- | --- |
| ProductCreated | Product | admin 등록 | audit | `product.events.v1` |
| ProductPriceChanged | Product | 가격 변경 | audit | 동 |
| ProductDiscontinued | Product | 단종 | audit + 카탈로그 invalidate | 동 |
| OrderCreated | Order | 주문 | audit | `product.order.events.v1` |
| OrderPaid | Order | 결제 승인 | 디지털 worker + 알림 + 정산 | 동 (consumer 분리) |
| OrderFulfilled | Order | 배송 / 다운로드 | 알림 | 동 |
| OrderCanceled | Order | 취소 | 재고 복원 + 알림 | 동 |
| PaymentApproved | Payment | /confirm 성공 | OrderPaid trigger | `product.payment.events.v1` |
| PaymentFailed | Payment | /confirm 실패 | 재고 복원 + 알림 | 동 |
| PaymentCanceled | Payment | 환불 | settlement adjust + 디지털 revoke | 동 |
| DigitalDeliveryReady | DigitalDelivery | 워터마크 완료 | 알림 + library 표시 | `product.digital.delivery.v1` |
| DigitalDeliveryRevoked | DigitalDelivery | 환불 | GDrive permission revoke | 동 |
| DigitalDeliveryFailed | DigitalDelivery | 워터마크 max 실패 | admin alert | 동 |
| InventoryDepleted | Inventory | 재고 0 | 카탈로그 SOLD_OUT | `product.inventory.events.v1` |
| InventoryRestocked | Inventory | 입고 | 알림 (재입고 알림 신청자) | 동 |
| ShippingDelivered | Shipment | 택배 완료 | OrderFulfilled trigger | 동 |

---

## 2. 공통 schema

```java
public sealed interface DomainEvent {
    String eventId();
    Instant occurredAt();
    String aggregateId();
    String aggregateType();
}

public record PaymentApproved(
    String eventId,
    Instant occurredAt,
    PaymentId paymentId,
    OrderId orderId,
    UserId buyerId,
    Money amount,
    String pgProvider
) implements DomainEvent {
    public String aggregateId() { return paymentId.value(); }
    public String aggregateType() { return "PAYMENT"; }
}
```

### 2.1 왜 eventId

- Kafka consumer dedup (idempotency).

### 2.2 왜 sealed interface

- pattern matching exhaustiveness (Java 21).
- 새 event 추가 시 컴파일러가 모든 handler 점검.

---

## 3. Listener 패턴 (F0~F9)

```java
@Component
class OrderPaidListener {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onOrderPaid(OrderPaid ev) {
        // digital worker enqueue / 알림 / 정산
    }
}
```

자세히: [[../implementation/payment-confirm-impl]].

---

## 4. Kafka 매핑 (F10+)

```java
@Component
class OrderPaidPublisher {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onOrderPaid(OrderPaid ev) {
        outbox.save(new OutboxRow(
            "product.order.events.v1",
            ev.aggregateId(),                  // partition key
            mapper.writeValueAsString(ev)));
    }
}
```

→ outbox worker 가 Kafka send.

자세히: [[../design-decisions/kafka-event-driven#4]].

---

## 5. 함정

### 함정 1 — 동기 listener (트랜잭션 안)
DB 락 / 외부 호출 실패 시 비즈니스 rollback.
→ AFTER_COMMIT.

### 함정 2 — eventId 없음
Kafka dedup X.
→ ULID.

### 함정 3 — schema 진화 없이 변경
backward-compat 깨짐.
→ version 별 record.

---

## 6. 관련

- [[domain-model|↑ hub]]
- [[product-aggregate]] / [[order-aggregate]] / [[payment-aggregate]] / [[digital-delivery-aggregate]]
- [[../design-decisions/kafka-event-driven]]
- [[../implementation/payment-confirm-impl]]
