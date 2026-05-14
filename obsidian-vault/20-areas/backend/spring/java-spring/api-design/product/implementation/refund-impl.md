---
title: "환불 구현 — PG cancel + revoke + 정산 adjust"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - refund
---

# 환불 구현 — PG cancel + revoke + 정산 adjust

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. API

```http
POST /api/v1/admin/payments/{paymentId}/cancel
Idempotency-Key: ...
{ "amount": 30000, "reason": "단순변심" }
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class RefundService {

    private final PaymentRepository payments;
    private final OrderRepository orders;
    private final RefundRepository refunds;
    private final SettlementRepository settlements;
    private final PaymentGatewayRouter router;
    private final PointService points;
    private final CouponService coupons;
    private final IdempotencyStore idempotency;
    private final ApplicationEventPublisher events;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public RefundResult cancel(String idempotencyKey, PaymentId paymentId,
                               Money refundAmount, String reason, UserId actor) {

        return idempotency.execute(idempotencyKey, () -> {

            var payment = payments.findById(paymentId).orElseThrow();
            var order = orders.findById(payment.orderId()).orElseThrow();

            // 디지털 검증 — 다운로드 전만 환불
            if (order.hasDigitalItem() && order.hasDownloadedDigital()) {
                throw new RefundNotAllowedException("digital already downloaded");
            }

            // 1. PG cancel API
            var pg = router.resolve(payment.pgProvider());
            var pgResult = pg.cancel(new PgCancelCommand(
                payment.pgPaymentKey(), refundAmount, reason));

            // 2. 도메인 적용
            var refundRecord = payment.cancel(refundAmount, reason, actor, clock.now());
            payments.save(payment);

            // 3. refund row
            var refund = new Refund(
                ids.next(), paymentId, refundAmount, reason,
                actor, "ADMIN", pgResult.cancelKey(),
                idempotencyKey, RefundStatus.COMPLETED, clock.now());
            refunds.save(refund);

            // 4. order 상태
            if (payment.status() == PaymentStatus.CANCELED) {
                order.refund(clock.now());
                orders.save(order);
            }

            // 5. 적립금 / 쿠폰 복원 (정책에 따라)
            points.restoreProrated(order.buyerId(), order.id(), refundAmount);
            // coupon.restore(...) — 정책 (기본 X)

            // 6. 정산 adjust
            settlements.addAdjustment(paymentId,
                "REFUND", refundAmount.negate(), reason, clock.now());

            // 7. event
            payment.events().forEach(events::publishEvent);

            return RefundResult.of(refund);
        });
    }
}
```

---

## 3. AFTER_COMMIT (디지털 revoke)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentCanceled(PaymentCanceled ev) {
    var deliveries = digitalDeliveryRepo.findByOrderId(ev.orderId());
    for (var d : deliveries) {
        d.revoke("REFUND", clock.now());
        digitalDeliveryRepo.save(d);
        // GDrive permission 제거 (외부 호출 — AFTER_COMMIT)
        gdrive.revokeUserAccess(d.storageFileId(), userEmail(d.userId()));
    }
}
```

자세히: [[digital-delivery-impl#revoke]].

---

## 4. 함정

### 함정 1 — PG cancel 후 DB rollback
state mismatch.
→ PG 호출 후 DB → 실패 시 alert.

### 함정 2 — 디지털 다운 후 환불 허용
부당 이득.
→ hasDownloadedDigital 검증.

### 함정 3 — 환불 합계 > payment.amount
도메인에서 검증.

### 함정 4 — 정산 adjust 누락
회계 mismatch.
→ settlement_adjustments.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/refund-policy]]
- [[../database/refunds-table]]
- [[../domain-model/payment-aggregate]]
- [[digital-delivery-impl]]
- [[settlement-impl]]
