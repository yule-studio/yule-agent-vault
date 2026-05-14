---
title: "Order Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:16:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - aggregate
  - order
---

# Order Aggregate

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. 구조

```
Order (root)
├── List<OrderItem>      ← entity (snapshot)
├── Address              ← VO (encrypted)
├── Money[s] (amounts)   ← VOs
└── coupon / point       ← 적용 record
```

---

## 2. Java

```java
public final class Order {
    private final OrderId id;
    private final String orderNumber;
    private final UserId buyerId;
    private OrderStatus status;
    private final List<OrderItem> items;
    private Money subtotal;
    private Money discount;
    private Money coupon;
    private Money point;
    private Money shipping;
    private Money tax;
    private Money total;
    private Address shippingAddress;
    private Instant requestedAt;
    private Instant paidAt;
    private Instant fulfilledAt;
    private Instant canceledAt;
    private String cancelReason;
    private long version;
    private final List<DomainEvent> events = new ArrayList<>();

    public static Order create(OrderId id, UserId buyerId, List<OrderItem> items,
                               Money shipping, Money tax,
                               Optional<CouponRedemption> coupon, Money pointUsed,
                               Address address, Instant now) {
        var o = new Order(id, generateOrderNumber(), buyerId, items, address, now);
        o.calculate(shipping, tax, coupon, pointUsed);
        o.status = OrderStatus.PENDING;
        o.events.add(new OrderCreated(id, buyerId, items, o.total, now));
        return o;
    }

    public void confirmPayment(Instant now) {
        if (status != OrderStatus.PENDING) throw new IllegalStateException();
        this.status = OrderStatus.PAID;
        this.paidAt = now;
        events.add(new OrderPaid(id, total, now));
    }

    public void fulfill(Instant now) {
        if (status != OrderStatus.PAID) throw new IllegalStateException();
        this.status = OrderStatus.FULFILLED;
        this.fulfilledAt = now;
        events.add(new OrderFulfilled(id, now));
    }

    public void cancel(String reason, Instant now) {
        if (status == OrderStatus.FULFILLED) throw new IllegalStateException("cannot cancel fulfilled");
        this.status = OrderStatus.CANCELED;
        this.canceledAt = now;
        this.cancelReason = reason;
        events.add(new OrderCanceled(id, reason, now));
    }

    public boolean isExpired(Duration timeout, Instant now) {
        return status == OrderStatus.PENDING
            && requestedAt.plus(timeout).isBefore(now);
    }

    private void calculate(...) { /* 가격 계산 */ }

    public Money totalAmount() { return total; }
}
```

---

## 3. 불변식

- amount split 합산 일치: `total = subtotal - discount - coupon - point + shipping + tax`.
- PENDING → PAID / CANCELED.
- FULFILLED 후 cancel 불가 (환불은 별도).
- items 1개 이상.

---

## 4. 도메인 이벤트

- OrderCreated
- OrderPaid
- OrderFulfilled
- OrderCanceled
- OrderRefunded (full refund)

---

## 5. 관련

- [[domain-model|↑ hub]]
- [[payment-aggregate]]
- [[../database/orders-table]]
- [[../enums/order-status]]
- [[../design-decisions/pricing-strategy]]
