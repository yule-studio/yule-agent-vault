---
title: "Payment Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:18:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - aggregate
  - payment
---

# Payment Aggregate

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. 구조

```
Payment (root)
└── List<RefundRecord>   ← entity (부분 환불 N개)
```

---

## 2. Java

```java
public final class Payment {
    private final PaymentId id;
    private final OrderId orderId;
    private final UserId buyerId;
    private final Money amount;
    private final PaymentMethod method;
    private PaymentStatus status;
    private String pgProvider;
    private String pgPaymentKey;
    private final List<RefundRecord> refunds = new ArrayList<>();
    private final Instant requestedAt;
    private Instant approvedAt;
    private Instant canceledAt;
    private long version;
    private final List<DomainEvent> events = new ArrayList<>();

    public static Payment initiate(PaymentId id, OrderId orderId, UserId buyerId,
                                   Money amount, PaymentMethod method,
                                   String pgProvider, Instant now) {
        return new Payment(id, orderId, buyerId, amount, method,
                           PaymentStatus.READY, pgProvider, now);
    }

    public void markInProgress(String pgPaymentKey, Instant now) {
        require(status == PaymentStatus.READY);
        this.pgPaymentKey = pgPaymentKey;
        this.status = PaymentStatus.IN_PROGRESS;
    }

    public void approve(Instant now) {
        if (status == PaymentStatus.DONE) return;        // idempotent
        require(status == PaymentStatus.IN_PROGRESS || status == PaymentStatus.READY);
        this.status = PaymentStatus.DONE;
        this.approvedAt = now;
        events.add(new PaymentApproved(id, orderId, amount, now));
    }

    public void fail(String reason, Instant now) {
        if (status == PaymentStatus.FAILED) return;
        this.status = PaymentStatus.FAILED;
        events.add(new PaymentFailed(id, orderId, reason, now));
    }

    public RefundRecord cancel(Money refundAmount, String reason, UserId actor, Instant now) {
        require(status == PaymentStatus.DONE || status == PaymentStatus.PARTIAL_CANCELED);
        var refunded = refunds.stream().map(RefundRecord::amount)
            .reduce(Money::add).orElse(Money.zero(amount.currency()));
        var remaining = amount.subtract(refunded);
        require(refundAmount.amountMinorUnit() <= remaining.amountMinorUnit(),
                "refund exceeds remaining");
        var rec = new RefundRecord(refundAmount, reason, actor, now);
        refunds.add(rec);
        var newTotal = refunded.add(refundAmount);
        this.status = newTotal.equals(amount) ? PaymentStatus.CANCELED
                                              : PaymentStatus.PARTIAL_CANCELED;
        this.canceledAt = now;
        events.add(new PaymentCanceled(id, orderId, refundAmount, reason, now));
        return rec;
    }

    public Money refundableAmount() {
        var refunded = refunds.stream().map(RefundRecord::amount)
            .reduce(Money::add).orElse(Money.zero(amount.currency()));
        return amount.subtract(refunded);
    }
}

public record RefundRecord(Money amount, String reason, UserId actor, Instant occurredAt) {}
```

---

## 3. 불변식

- amount 변경 X (immutable).
- refunded 합계 ≤ amount.
- status 전이 표 (PaymentStatus enum 참고).
- approve idempotent (DONE → DONE = no-op).

---

## 4. 이벤트

- PaymentApproved
- PaymentFailed
- PaymentCanceled (refund)

---

## 5. 관련

- [[domain-model|↑ hub]]
- [[order-aggregate]]
- [[../enums/payment-status]]
- [[../database/payments-table]]
- [[../database/refunds-table]]
- [[../design-decisions/refund-policy]]
