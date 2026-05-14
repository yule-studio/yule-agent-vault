---
title: "결제 confirm 구현 ★ — Idempotency + amount 검증 + PG 호출"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - payment
---

# 결제 confirm 구현 ★ — Idempotency + amount 검증 + PG 호출

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

> 결제 흐름의 가장 중요한 endpoint — Idempotency / amount 변조 방어 / PG 호출 / 멱등 / outbox.

---

## 1. Controller

```java
@RestController
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentConfirmService service;

    @PostMapping("/api/v1/payments/confirm")
    public ResponseEntity<PaymentConfirmResponse> confirm(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody PaymentConfirmRequest req,
            @AuthenticationPrincipal AuthUser auth) {

        var result = service.confirm(
            idempotencyKey,
            req.paymentKey(),
            new OrderId(req.orderId()),
            Money.krw(req.amount()),
            auth.userId());

        return ResponseEntity.ok(PaymentConfirmResponse.from(result));
    }
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class PaymentConfirmService {

    private final OrderRepository orders;
    private final PaymentRepository payments;
    private final PaymentTransactionRepository transactions;
    private final PaymentGatewayRouter router;
    private final IdempotencyStore idempotency;
    private final ApplicationEventPublisher events;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public PaymentResult confirm(String idempotencyKey, String paymentKey,
                                 OrderId orderId, Money amount, UserId buyerId) {

        return idempotency.execute(idempotencyKey, () -> {

            // 1. order 조회 + 검증
            var order = orders.findById(orderId).orElseThrow();
            if (!order.buyerId().equals(buyerId))
                throw new ForbiddenException();
            if (!order.totalAmount().equals(amount))
                throw new AmountTamperedException(amount, order.totalAmount());

            // 2. payment 조회
            var payment = payments.findByOrderId(orderId).orElseThrow();
            if (payment.status() == PaymentStatus.DONE)
                return PaymentResult.of(payment);   // idempotent (already confirmed)

            // 3. PG 호출
            var pg = router.resolve(payment.pgProvider());
            var start = clock.now();
            try {
                var pgResult = pg.confirm(new PgConfirmCommand(
                    paymentKey, order.orderNumber(), amount));

                // 4. 도메인 적용
                payment.markInProgress(paymentKey, clock.now());
                payment.approve(clock.now());
                order.confirmPayment(clock.now());

                payments.save(payment);
                orders.save(order);

                // 5. audit
                transactions.save(PaymentTransaction.success(
                    ids.next(), payment.id(), "CONFIRM",
                    payment.pgProvider(), pgResult,
                    Duration.between(start, clock.now()).toMillis(),
                    clock.now()));

                // 6. event (AFTER_COMMIT)
                payment.events().forEach(events::publishEvent);
                order.events().forEach(events::publishEvent);

                return PaymentResult.of(payment);

            } catch (PgException e) {
                payment.fail(e.getMessage(), clock.now());
                payments.save(payment);
                transactions.save(PaymentTransaction.failure(
                    ids.next(), payment.id(), "CONFIRM",
                    payment.pgProvider(), e.getMessage(),
                    Duration.between(start, clock.now()).toMillis(),
                    clock.now()));
                throw e;
            }
        });
    }
}
```

---

## 3. AFTER_COMMIT Listeners

```java
@Component
@RequiredArgsConstructor
public class PaymentApprovedListeners {

    private final OrderRepository orders;
    private final InventoryService inventory;
    private final DigitalDeliveryService digital;
    private final NotificationOutbox notifications;
    private final ApplicationEventPublisher events;
    private final OutboxRepository outbox;     // F10+ Kafka 용

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onPaymentApproved(PaymentApproved ev) {
        // F0~F9: in-process
        var order = orders.findById(ev.orderId()).orElseThrow();

        // 디지털 worker enqueue (BOOK / DIGITAL 만)
        order.items().stream()
            .filter(i -> i.productType() != ProductType.PHYSICAL)
            .forEach(i -> digital.start(order.id(), i.id(), ev.buyerId()));

        // 알림
        notifications.send(ev.buyerId(), "PAYMENT_APPROVED",
            Map.of("amount", ev.amount(), "orderNumber", order.orderNumber()));

        // F10+ Kafka: outbox 도 INSERT
        outbox.save(OutboxEvent.of(
            "product.payment.events.v1",
            ev.orderId().value(),
            ev));
    }
}
```

자세히: [[../design-decisions/kafka-event-driven#4]].

---

## 4. PG 어댑터 (Toss)

```java
@Component
public class TossPaymentGateway implements PaymentGateway {

    private final RestTemplate http;
    private final TossProperties props;

    public String provider() { return "TOSS"; }

    public PgConfirmResult confirm(PgConfirmCommand cmd) {
        var headers = new HttpHeaders();
        headers.setBasicAuth(props.secretKey(), "");
        headers.setContentType(MediaType.APPLICATION_JSON);

        var body = Map.of(
            "paymentKey", cmd.paymentKey(),
            "orderId", cmd.orderId(),
            "amount", cmd.amount().amountMinorUnit());

        try {
            var resp = http.exchange(
                props.baseUrl() + "/v1/payments/confirm",
                HttpMethod.POST,
                new HttpEntity<>(body, headers),
                TossPaymentResponse.class);

            return PgConfirmResult.success(resp.getBody());

        } catch (HttpStatusCodeException e) {
            var error = parse(e.getResponseBodyAsString());
            throw new PgException(error.code(), error.message());
        }
    }

    public PgCancelResult cancel(PgCancelCommand cmd) { /* ... */ }
    public boolean verifyWebhookSignature(WebhookRequest req) { /* ... */ }
}
```

---

## 5. 함정

### 함정 1 — amount 검증 안 함
변조.
→ order.totalAmount 비교.

### 함정 2 — Idempotency-Key 헤더 무시
중복 결제.
→ store.execute.

### 함정 3 — PG 호출이 트랜잭션 밖
DB rollback 후 PG 가 승인 — state mismatch.
→ 트랜잭션 안 + webhook 보강.

### 함정 4 — PG timeout 30s
사용자 화면 stuck.
→ HTTP timeout 5s + webhook 보강.

### 함정 5 — PG response raw 안 저장
분쟁 evidence X.
→ payment_transactions.

### 함정 6 — events 발행 위치
@Transactional 안의 동기 listener 시 외부 호출 fail 시 rollback.
→ AFTER_COMMIT.

---

## 6. 관련

- [[implementation|↑ hub]]
- [[payment-init-impl]]
- [[payment-webhook-impl]]
- [[../design-decisions/payment-flow]]
- [[../security/idempotency-key]]
- [[../database/payments-table]]
- [[../database/payment-transactions-table]]
- [[../design-decisions/kafka-event-driven]]
