---
title: "결제 (PG 연동 + Webhook) — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - commerce
  - payment
  - webhook
---

# 결제 (PG 연동 + Webhook) — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 토스/카카오 PG / Idempotency / webhook 서명 검증 / 환불 |

**[[api-design|↑ api-design hub]]**

> 전제: [[order-stock]]. `orders`, `order_items` 존재.
> **참고 PG**: 토스페이먼츠 (가장 표준적), 카카오페이, 네이버페이, NHN KCP, 이니시스.
> 본 레시피는 **토스페이먼츠** 식 (현대적 REST API + webhook + Idempotency 헤더) 기준.

---

## 1. 무엇을 만드는가

### 1.1 결제 흐름 (가장 표준)

```
1. [클라] 결제수단 선택 → PG SDK 로 결제창 열기 (PG 호스팅)
2. [PG]  사용자 결제 정보 입력 (카드 / 계좌 / 페이) → 성공 callback URL 로 redirect
3. [클라] /api/v1/payments/confirm 호출 (paymentKey + orderId + amount)
4. [서버] PG 의 /confirm API 호출 → 승인 처리
5. [서버] order.confirmPayment() + 결제 row 저장 + 영수증 발급
6. [PG]  비동기 webhook 발송 (보강 — 누락 시 서버가 polling 으로 보완 가능)
```

### 1.2 API

```
POST /api/v1/payments/confirm          # 클라이언트가 callback 시 호출
POST /api/v1/payments/{id}/cancel      # 부분 / 전체 환불
GET  /api/v1/payments/{id}             # 결제 상세
POST /api/v1/payments/webhooks/toss    # PG 가 호출
```

### 1.3 비기능

- **`Idempotency-Key`** — PG 호출 / DB 저장 모두 멱등
- **webhook 서명 검증** — HMAC-SHA-256
- **상태 머신** — `READY → IN_PROGRESS → DONE / FAILED / CANCELED / PARTIAL_CANCELED`
- **결제 row 와 주문 row 트랜잭션 일관성** — 한 RDB
- **PII 마스킹** — 카드번호 / CVC 절대 저장 X (PCI-DSS)
- **결제 금액 검증** — 클라이언트가 보낸 amount 와 order.totalAmount 일치 확인 (변조 방지)

---

## 2. 도메인 모델

### 2.1 Payment Aggregate

```java
// src/main/java/com/example/shop/domain/payment/Payment.java
public final class Payment {

    private final PaymentId id;
    private final OrderId orderId;
    private final UserId buyerId;
    private final Money amount;
    private final PaymentMethod method;
    private PaymentStatus status;
    private String pgPaymentKey;         // PG 가 발급한 키 (예: tossPaymentKey)
    private final Instant requestedAt;
    private Instant approvedAt;
    private Instant canceledAt;
    private final List<RefundRecord> refunds = new ArrayList<>();
    private final List<DomainEvent> events = new ArrayList<>();

    private Payment(PaymentId id, OrderId orderId, UserId buyerId, Money amount,
                    PaymentMethod method, PaymentStatus status, Instant now) {
        this.id = id; this.orderId = orderId; this.buyerId = buyerId;
        this.amount = amount; this.method = method;
        this.status = status; this.requestedAt = now;
    }

    public static Payment initiate(PaymentId id, OrderId orderId, UserId buyerId,
                                   Money amount, PaymentMethod method, Instant now) {
        return new Payment(id, orderId, buyerId, amount, method, PaymentStatus.READY, now);
    }

    public void markInProgress(String pgPaymentKey, Instant now) {
        if (status != PaymentStatus.READY) throw new IllegalStateException(status.name());
        this.status = PaymentStatus.IN_PROGRESS;
        this.pgPaymentKey = pgPaymentKey;
    }

    public void approve(Instant now) {
        if (status == PaymentStatus.DONE) return;     // 멱등 — webhook retry 안전
        if (status != PaymentStatus.IN_PROGRESS && status != PaymentStatus.READY)
            throw new IllegalStateException("cannot approve from " + status);
        this.status = PaymentStatus.DONE;
        this.approvedAt = now;
        events.add(new PaymentApproved(id, orderId, amount, now));
    }

    public void fail(String reason, Instant now) {
        if (status == PaymentStatus.FAILED) return;
        this.status = PaymentStatus.FAILED;
        events.add(new PaymentFailed(id, orderId, reason, now));
    }

    public RefundRecord cancel(Money refundAmount, String reason, Instant now) {
        if (status != PaymentStatus.DONE && status != PaymentStatus.PARTIAL_CANCELED)
            throw new IllegalStateException("cannot cancel from " + status);
        var totalRefunded = refunds.stream().map(RefundRecord::amount).reduce(Money::add)
            .orElse(Money.krw(0));
        var remaining = amount.subtract(totalRefunded);
        if (refundAmount.amountMinorUnit() > remaining.amountMinorUnit())
            throw new IllegalArgumentException("refund exceeds remaining");
        var refund = new RefundRecord(refundAmount, reason, now);
        refunds.add(refund);
        var newTotal = totalRefunded.add(refundAmount);
        this.status = newTotal.equals(amount) ? PaymentStatus.CANCELED : PaymentStatus.PARTIAL_CANCELED;
        this.canceledAt = now;
        events.add(new PaymentCanceled(id, orderId, refundAmount, reason, now));
        return refund;
    }

    public boolean amountMatches(Money expected) { return amount.equals(expected); }

    // getters omitted
}

public enum PaymentStatus { READY, IN_PROGRESS, DONE, FAILED, CANCELED, PARTIAL_CANCELED }
public enum PaymentMethod { CARD, BANK_TRANSFER, MOBILE, EASY_PAY }

public record RefundRecord(Money amount, String reason, Instant occurredAt) {}
```

### 2.2 Domain Events

```java
public record PaymentApproved(PaymentId paymentId, OrderId orderId, Money amount, Instant occurredAt) implements DomainEvent {}
public record PaymentFailed(PaymentId paymentId, OrderId orderId, String reason, Instant occurredAt) implements DomainEvent {}
public record PaymentCanceled(PaymentId paymentId, OrderId orderId, Money refundAmount, String reason, Instant occurredAt) implements DomainEvent {}
```

### 2.3 Repository

```java
public interface PaymentRepository {
    Payment save(Payment p);
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId orderId);
    Optional<Payment> findByPgPaymentKey(String key);
}
```

---

## 3. DB 스키마

```sql
-- V50__create_payments.sql
CREATE TABLE payments (
  id              CHAR(26) PRIMARY KEY,
  order_id        CHAR(26) NOT NULL REFERENCES orders(id),
  buyer_id        CHAR(26) NOT NULL REFERENCES users(id),
  amount_krw      BIGINT NOT NULL CHECK (amount_krw >= 0),
  method          VARCHAR(20) NOT NULL,
  status          VARCHAR(20) NOT NULL,
  pg_provider     VARCHAR(20) NOT NULL,                -- TOSS / KAKAO / ...
  pg_payment_key  VARCHAR(100),
  version         BIGINT NOT NULL DEFAULT 0,
  requested_at    TIMESTAMPTZ NOT NULL,
  approved_at     TIMESTAMPTZ,
  canceled_at     TIMESTAMPTZ
);
CREATE UNIQUE INDEX ux_payments_order ON payments (order_id);
CREATE UNIQUE INDEX ux_payments_pg_key
  ON payments (pg_provider, pg_payment_key) WHERE pg_payment_key IS NOT NULL;
CREATE INDEX ix_payments_status_requested ON payments (status, requested_at DESC);

-- V51__create_payment_refunds.sql
CREATE TABLE payment_refunds (
  id           CHAR(26) PRIMARY KEY,
  payment_id   CHAR(26) NOT NULL REFERENCES payments(id),
  amount_krw   BIGINT NOT NULL CHECK (amount_krw > 0),
  reason       VARCHAR(500),
  pg_cancel_key VARCHAR(100),
  occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_payment_refunds_payment ON payment_refunds (payment_id);

-- V52__create_payment_webhooks.sql — webhook 수신 기록 + 멱등
CREATE TABLE payment_webhooks (
  id           CHAR(26) PRIMARY KEY,
  provider     VARCHAR(20) NOT NULL,
  event_id     VARCHAR(100) NOT NULL,            -- PG 발급 이벤트 ID
  payment_key  VARCHAR(100),
  event_type   VARCHAR(50) NOT NULL,
  payload      JSONB NOT NULL,
  signature    VARCHAR(200) NOT NULL,
  received_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at TIMESTAMPTZ,
  process_result VARCHAR(20)                       -- OK / FAILED / IGNORED
);
CREATE UNIQUE INDEX ux_payment_webhooks_event ON payment_webhooks (provider, event_id);
CREATE INDEX ix_payment_webhooks_unprocessed
  ON payment_webhooks (received_at) WHERE processed_at IS NULL;
```

> **핵심**: `payment_webhooks.event_id` UNIQUE — PG retry 시 중복 처리 차단.

---

## 4. PG Client 추상화

### 4.1 Port

```java
// src/main/java/com/example/shop/domain/payment/PgClient.java
public interface PgClient {
    PgApprovalResult confirm(PgConfirmRequest req);
    PgCancelResult cancel(PgCancelRequest req);
    PgVerifyResult verify(String paymentKey);          // 폴링 보완

    record PgConfirmRequest(String paymentKey, String orderId, long amountKrw) {}
    record PgApprovalResult(boolean approved, String pgKey, long amountKrw,
                            String method, String failureReason) {}
    record PgCancelRequest(String paymentKey, long cancelAmount, String reason) {}
    record PgCancelResult(boolean ok, String cancelKey, String failureReason) {}
    record PgVerifyResult(String status, long amountKrw) {}
}
```

### 4.2 Toss 구현 — Resilience4j 로 회로차단 / 재시도

```java
// src/main/java/com/example/shop/infrastructure/external/pg/TossPgClient.java
@Component
public class TossPgClient implements PgClient {

    private final RestClient http;
    private final String secretKey;

    public TossPgClient(RestClient.Builder builder,
                        @Value("${app.pg.toss.secret-key}") String secretKey,
                        @Value("${app.pg.toss.base-url}") String baseUrl) {
        this.http = builder.baseUrl(baseUrl)
            .defaultHeaders(h -> h.setBasicAuth(
                Base64.getEncoder().encodeToString((secretKey + ":").getBytes())))
            .requestFactory(simpleClientHttpRequestFactory(Duration.ofSeconds(5)))
            .build();
        this.secretKey = secretKey;
    }

    @CircuitBreaker(name = "pg-toss", fallbackMethod = "confirmFallback")
    @Retry(name = "pg-toss")
    @Override
    public PgApprovalResult confirm(PgConfirmRequest req) {
        var body = Map.of(
            "paymentKey", req.paymentKey(),
            "orderId",    req.orderId(),
            "amount",     req.amountKrw()
        );
        try {
            var res = http.post().uri("/v1/payments/confirm")
                .header("Idempotency-Key", req.paymentKey())     // PG 도 멱등 헤더 지원
                .contentType(MediaType.APPLICATION_JSON)
                .body(body)
                .retrieve()
                .body(Map.class);
            return new PgApprovalResult(
                true,
                (String) res.get("paymentKey"),
                ((Number) res.get("totalAmount")).longValue(),
                (String) res.get("method"),
                null
            );
        } catch (HttpStatusCodeException e) {
            var msg = e.getResponseBodyAsString();
            return new PgApprovalResult(false, null, 0, null, msg);
        }
    }

    private PgApprovalResult confirmFallback(PgConfirmRequest req, Throwable t) {
        // 회로 OPEN 상태 — 빠른 실패. 사용자에게 재시도 안내.
        return new PgApprovalResult(false, null, 0, null,
            "PG temporarily unavailable: " + t.getMessage());
    }

    // cancel, verify 유사
}
```

```yaml
# application.yml
app:
  pg:
    toss:
      base-url: https://api.tosspayments.com
      secret-key: ${TOSS_SECRET_KEY}        # TODO(secret): vault
      webhook-secret: ${TOSS_WEBHOOK_SECRET}

resilience4j:
  circuitbreaker:
    instances:
      pg-toss:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 20
  retry:
    instances:
      pg-toss:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
```

> **함정**: PG 호출이 동기 + 트랜잭션 안에서면 락 hold 시간 폭증. **`@Transactional` 밖에서 호출 → 결과를 기준으로 트랜잭션 시작**.

---

## 5. ConfirmPaymentUseCase (핵심 흐름)

```java
// src/main/java/com/example/shop/application/payment/ConfirmPaymentUseCase.java
@Service
public class ConfirmPaymentUseCase {

    private final OrderRepository orders;
    private final PaymentRepository payments;
    private final PgClient pg;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;
    private final TransactionTemplate tx;

    public ConfirmPaymentUseCase(OrderRepository orders, PaymentRepository payments,
                                 PgClient pg, IdGenerator ids, Clock clock,
                                 ApplicationEventPublisher events,
                                 TransactionTemplate tx) {
        this.orders = orders; this.payments = payments; this.pg = pg;
        this.ids = ids; this.clock = clock; this.events = events; this.tx = tx;
    }

    public ConfirmResult handle(ConfirmPaymentCommand cmd) {
        // 1. 사전 검증 (트랜잭션 밖)
        var order = orders.findById(cmd.orderId()).orElseThrow(NotFoundException::new);
        if (!order.isOwnedBy(cmd.buyerId())) throw new ForbiddenException();
        if (order.status() != OrderStatus.PENDING_PAYMENT)
            throw new IllegalStateException("order status: " + order.status());
        // 금액 변조 방어
        if (cmd.amount() != order.totalAmount().amountMinorUnit())
            throw new PaymentAmountMismatchException(cmd.amount(), order.totalAmount());

        // 2. 기존 결제 row 가 있나? (멱등)
        var existing = payments.findByOrderId(cmd.orderId());
        Payment payment;
        if (existing.isPresent() && existing.get().status() == PaymentStatus.DONE) {
            return new ConfirmResult(existing.get().id(), "DONE");      // idempotent
        }
        payment = existing.orElseGet(() -> Payment.initiate(
            new PaymentId(ids.next()), cmd.orderId(), cmd.buyerId(),
            order.totalAmount(),
            cmd.method() == null ? PaymentMethod.CARD : cmd.method(),
            Instant.now(clock)
        ));
        payment.markInProgress(cmd.pgPaymentKey(), Instant.now(clock));
        tx.executeWithoutResult(s -> payments.save(payment));

        // 3. PG 호출 (트랜잭션 밖!)
        var result = pg.confirm(new PgClient.PgConfirmRequest(
            cmd.pgPaymentKey(), cmd.orderId().value(), cmd.amount()
        ));

        // 4. 결과 반영 (트랜잭션 안)
        if (!result.approved()) {
            tx.executeWithoutResult(s -> {
                payment.fail(result.failureReason(), Instant.now(clock));
                payments.save(payment);
            });
            payment.pullDomainEvents().forEach(events::publishEvent);
            throw new PaymentApprovalFailedException(result.failureReason());
        }

        // PG 응답 amount 와 주문 amount 재검증 (이중 방어)
        if (result.amountKrw() != order.totalAmount().amountMinorUnit())
            throw new PaymentAmountMismatchException(result.amountKrw(), order.totalAmount());

        tx.executeWithoutResult(s -> {
            payment.approve(Instant.now(clock));
            payments.save(payment);

            // 주문도 같이 확정 — 같은 트랜잭션
            order.confirmPayment(Instant.now(clock));
            orders.save(order);
        });

        payment.pullDomainEvents().forEach(events::publishEvent);
        order.pullDomainEvents().forEach(events::publishEvent);
        return new ConfirmResult(payment.id(), "DONE");
    }
}

public record ConfirmPaymentCommand(
    OrderId orderId, UserId buyerId, String pgPaymentKey, long amount, PaymentMethod method
) {}
public record ConfirmResult(PaymentId paymentId, String status) {}
```

### 5.1 왜 PG 호출을 트랜잭션 밖에서?

```
❌ 안티
@Transactional
public void handle(...) {
    var order = orders.findById(...);
    pg.confirm(...);                 // 5초 걸리면 DB 락 5초 hold
    payments.save(...);
    orders.save(...);
}
```

→ 동시 트래픽 = DB connection pool 고갈.

```
✅ 올바른
1. (tx) 결제 row 생성 / 검증 / 락
2. (no tx) PG 호출 — 외부 IO
3. (tx) 결과 반영
```

---

## 6. Webhook 수신 — 보강 / 안전망

### 6.1 Webhook 흐름

```
PG 가 결제 완료 후 비동기로 webhook 발송
   ↓
   POST /api/v1/payments/webhooks/toss
   Headers: TossPayments-Signature: <hmac>
   Body: {"eventId":"...","paymentKey":"...","status":"DONE","amount":98000,...}
   ↓
서버:
   1. 서명 검증 (HMAC-SHA-256, secret 은 PG 가 발급)
   2. payment_webhooks.event_id UNIQUE — 중복이면 200 OK 즉시 반환 (멱등)
   3. payment_webhooks 에 raw 저장
   4. 별도 워커 / inline 처리 — Payment 상태 보강
```

### 6.2 Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/payment/PaymentWebhookController.java
@RestController
@RequestMapping("/api/v1/payments/webhooks")
public class PaymentWebhookController {

    private final WebhookProcessor processor;
    private final String webhookSecret;

    public PaymentWebhookController(WebhookProcessor processor,
                                    @Value("${app.pg.toss.webhook-secret}") String secret) {
        this.processor = processor; this.webhookSecret = secret;
    }

    @PostMapping("/toss")
    public ResponseEntity<Void> toss(@RequestBody String rawBody,
                                     @RequestHeader("TossPayments-Signature") String signature) {
        // 1. 서명 검증
        var expected = hmacSha256Hex(rawBody, webhookSecret);
        if (!MessageDigest.isEqual(expected.getBytes(), signature.getBytes()))
            throw new BadRequestException("invalid signature");

        // 2. 멱등 + 보강 처리
        processor.process("TOSS", rawBody, signature);

        // PG 가 200 만 보면 retry 안 함 — 즉시 200
        return ResponseEntity.ok().build();
    }

    private static String hmacSha256Hex(String body, String secret) {
        try {
            var mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            var bytes = mac.doFinal(body.getBytes(StandardCharsets.UTF_8));
            var sb = new StringBuilder();
            for (var b : bytes) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (Exception e) { throw new IllegalStateException(e); }
    }
}
```

> **함정**: `String.equals` 로 서명 비교 = timing attack 위험. **`MessageDigest.isEqual`** (constant-time).

### 6.3 WebhookProcessor

```java
@Service
public class WebhookProcessor {

    private final WebhookRepository repo;
    private final PaymentRepository payments;
    private final OrderRepository orders;
    private final ObjectMapper json;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public void process(String provider, String rawBody, String signature) {
        try {
            var node = json.readTree(rawBody);
            var eventId = node.get("eventId").asText();
            var paymentKey = node.has("paymentKey") ? node.get("paymentKey").asText() : null;
            var eventType = node.get("eventType").asText();

            // 멱등 — DB unique 가 진실의 원천
            try {
                repo.save(new WebhookRow(
                    ids.next(), provider, eventId, paymentKey, eventType,
                    rawBody, signature, Instant.now(clock), null, null
                ));
            } catch (DataIntegrityViolationException e) {
                // 이미 처리됨 — 무시
                return;
            }

            // 보강 — 우리 DB 의 결제 상태와 PG 의 webhook 일치 확인
            if (paymentKey != null) {
                var maybePayment = payments.findByPgPaymentKey(paymentKey);
                if (maybePayment.isPresent()) {
                    var p = maybePayment.get();
                    switch (eventType) {
                        case "PAYMENT_STATUS_CHANGED" -> {
                            var pgStatus = node.get("status").asText();
                            if ("DONE".equals(pgStatus) && p.status() != PaymentStatus.DONE) {
                                // confirm 미처리 — 보강 처리
                                p.approve(Instant.now(clock));
                                payments.save(p);
                                var order = orders.findById(p.orderId()).orElseThrow();
                                if (order.status() == OrderStatus.PENDING_PAYMENT) {
                                    order.confirmPayment(Instant.now(clock));
                                    orders.save(order);
                                }
                                p.pullDomainEvents().forEach(events::publishEvent);
                                order.pullDomainEvents().forEach(events::publishEvent);
                            }
                        }
                        // 환불 webhook 등 다른 case
                    }
                }
            }

            repo.markProcessed(eventId, "OK", Instant.now(clock));

        } catch (Exception e) {
            repo.markProcessed(extractEventId(rawBody), "FAILED", Instant.now(clock));
            // 추후 재시도 또는 알림
        }
    }
}
```

> **보강의 이유**: confirm 단계에서 네트워크 끊김 / 서버 죽음 → 결제는 됐는데 DB 미반영. webhook 이 안전망.

---

## 7. 환불 (CancelPaymentUseCase)

```java
@Service
public class CancelPaymentUseCase {

    private final PaymentRepository payments;
    private final OrderRepository orders;
    private final PgClient pg;
    private final Clock clock;
    private final TransactionTemplate tx;
    private final ApplicationEventPublisher events;

    public CancelResult handle(PaymentId id, Money refundAmount, String reason,
                               UserId requester, boolean isAdmin) {
        var payment = payments.findById(id).orElseThrow(NotFoundException::new);
        if (!payment.isOwnedBy(requester) && !isAdmin) throw new ForbiddenException();

        // PG 호출 (tx 밖)
        var result = pg.cancel(new PgClient.PgCancelRequest(
            payment.pgPaymentKey(), refundAmount.amountMinorUnit(), reason
        ));
        if (!result.ok()) throw new RefundFailedException(result.failureReason());

        // 결과 반영 (tx)
        tx.executeWithoutResult(s -> {
            payment.cancel(refundAmount, reason, Instant.now(clock));
            payments.save(payment);
            // 전체 환불이면 주문도 취소 + 재고 복구 (order-stock 의 CancelOrderUseCase 호출 또는 이벤트)
            if (payment.status() == PaymentStatus.CANCELED) {
                var order = orders.findById(payment.orderId()).orElseThrow();
                order.cancel("payment refunded", Instant.now(clock));
                orders.save(order);
                order.pullDomainEvents().forEach(events::publishEvent);
                // 재고 복구는 OrderCanceled listener 가 처리
            }
        });
        payment.pullDomainEvents().forEach(events::publishEvent);
        return new CancelResult(payment.id(), payment.status().name());
    }
}
```

---

## 8. Controller (confirm / cancel)

```java
@RestController
@RequestMapping("/api/v1/payments")
@PreAuthorize("isAuthenticated()")
public class PaymentController {

    private final ConfirmPaymentUseCase confirm;
    private final CancelPaymentUseCase cancel;
    private final GetPaymentUseCase get;

    @PostMapping("/confirm")
    public ApiResponse<ConfirmResult> confirm(
        @Valid @RequestBody ConfirmRequest req,
        Authentication auth
    ) {
        var r = confirm.handle(new ConfirmPaymentCommand(
            new OrderId(req.orderId()),
            new UserId(auth.getName()),
            req.paymentKey(),
            req.amount(),
            req.method() == null ? null : PaymentMethod.valueOf(req.method())
        ));
        return ApiResponse.ok(r);
    }

    @PostMapping("/{id}/cancel")
    public ApiResponse<CancelResult> cancel(
        @PathVariable String id,
        @Valid @RequestBody CancelRequest req,
        Authentication auth
    ) {
        return ApiResponse.ok(cancel.handle(
            new PaymentId(id),
            Money.krw(req.amount()),
            req.reason(),
            new UserId(auth.getName()),
            hasAdminRole(auth)
        ));
    }

    @GetMapping("/{id}")
    public ApiResponse<PaymentDetailResponse> get(@PathVariable String id, Authentication auth) {
        return ApiResponse.ok(get.handle(new PaymentId(id), new UserId(auth.getName())));
    }
}

public record ConfirmRequest(
    @NotBlank String orderId,
    @NotBlank String paymentKey,
    @Positive long amount,
    String method
) {}
public record CancelRequest(@Positive long amount, @NotBlank String reason) {}
```

---

## 9. 테스트

### 9.1 멱등 — 같은 confirm 두 번

```java
@Test
void confirm_is_idempotent_on_second_call() {
    var order = orderFixture.placeOrder(amount = 98000);
    pgFixture.willApprove("tossKey-1", 98000);

    var r1 = service.handle(new ConfirmPaymentCommand(
        order.id(), order.buyerId(), "tossKey-1", 98000, PaymentMethod.CARD));
    var r2 = service.handle(new ConfirmPaymentCommand(
        order.id(), order.buyerId(), "tossKey-1", 98000, PaymentMethod.CARD));

    assertThat(r1.paymentId()).isEqualTo(r2.paymentId());
    assertThat(r2.status()).isEqualTo("DONE");
    verify(pg, times(1)).confirm(any());      // PG 는 한 번만
}
```

### 9.2 금액 변조 거부

```java
@Test
void confirm_rejects_amount_tampering() {
    var order = orderFixture.placeOrder(amount = 98000);
    assertThatThrownBy(() -> service.handle(new ConfirmPaymentCommand(
        order.id(), order.buyerId(), "tossKey-1", 9800, PaymentMethod.CARD     // 1/10 시도
    ))).isInstanceOf(PaymentAmountMismatchException.class);
}
```

### 9.3 Webhook 서명 검증

```java
@Test
void webhook_rejects_invalid_signature() {
    var body = """{"eventId":"e1","paymentKey":"k","eventType":"PAYMENT_STATUS_CHANGED","status":"DONE"}""";
    var res = rest.exchange("/api/v1/payments/webhooks/toss",
        HttpMethod.POST,
        new HttpEntity<>(body, Map.of("TossPayments-Signature", "INVALID")),
        Void.class);
    assertThat(res.getStatusCode().value()).isEqualTo(400);
}

@Test
void webhook_is_idempotent_on_same_event_id() {
    var body = """...""";
    var sig = correctSignature(body);
    rest.exchange("/api/v1/payments/webhooks/toss", HttpMethod.POST,
        new HttpEntity<>(body, Map.of("TossPayments-Signature", sig)), Void.class);
    rest.exchange("/api/v1/payments/webhooks/toss", HttpMethod.POST,
        new HttpEntity<>(body, Map.of("TossPayments-Signature", sig)), Void.class);
    var webhooks = webhookRepo.findAll();
    assertThat(webhooks).hasSize(1);
}
```

---

## 10. 운영 체크리스트

- [ ] PG `secret-key` / `webhook-secret` 은 vault 주입
- [ ] webhook endpoint 는 인증 없이 (공개) — IP 화이트리스트 또는 서명만 의존
- [ ] 서명 검증 `MessageDigest.isEqual` (constant-time)
- [ ] `payment_webhooks.event_id` UNIQUE — 멱등
- [ ] `confirm` 의 amount 검증 (DB order.totalAmount 와 일치)
- [ ] PG 호출은 트랜잭션 밖
- [ ] Resilience4j 회로차단 / retry / timeout
- [ ] PG status 와 우리 DB status 의 정합성 점검 job (매일)
- [ ] PCI-DSS — 카드번호 / CVC 절대 저장 X (PG 호스팅에 위임)
- [ ] 환불 정책 — 부분 환불 가능 / 한도
- [ ] webhook 받기 endpoint timeout < 3초 (PG 가 보통 5초 안에 응답 기대)
- [ ] webhook retry 자동 — 우리 DB 의 `process_result=FAILED` 인 것 재처리
- [ ] 결제 / 환불 audit log (`payment_webhooks`, `stock_movements`)

---

## 11. 함정 모음

### 함정 1 — 금액 검증 누락
클라이언트가 보낸 amount 만 믿고 PG 호출. 변조하면 1원으로 결제 가능. **DB order.totalAmount 가 진실의 원천**.

### 함정 2 — PG 호출을 트랜잭션 안에서
DB 락 5~10초 hold. connection pool 고갈. **트랜잭션 밖**.

### 함정 3 — webhook 서명 미검증
누구나 위조 webhook 보내서 주문 완료 처리. **항상 HMAC 검증**.

### 함정 4 — 서명 비교 `equals`
timing attack. **`MessageDigest.isEqual`**.

### 함정 5 — webhook event_id 멱등 X
PG retry 시 결제 두 번 처리. **DB UNIQUE 또는 Redis SETNX**.

### 함정 6 — 결제 / 주문 분리 트랜잭션
payment DONE + order PENDING_PAYMENT 상태 따로 굳음. **같은 트랜잭션**.

### 함정 7 — PG status 와 우리 DB 불일치 미점검
PG 는 DONE 인데 우리 DB 는 IN_PROGRESS — confirm 누락. **매일 점검 job** (`PG.list 결제` vs `payments.status`).

### 함정 8 — 카드번호 / CVC DB 저장
PCI-DSS 위반 + 사고. **절대 X. PG 호스팅에 위임**.

### 함정 9 — 환불 한도 검증 누락
원금 + 누적 환불 > 결제 금액 가능. **`payment.cancel()` 도메인에서 검증**.

### 함정 10 — webhook 받는 endpoint 가 인증 필요
PG 는 인증 없이 호출. **공개 endpoint + 서명만 의존**. (CSRF token 등도 X)

### 함정 11 — webhook timeout 너무 김
PG 가 5초 내 응답 못 받으면 retry. webhook handler 가 동기로 다 처리하면 retry storm. **수신 + 200 즉시 반환 → 별도 워커가 무거운 처리**.

### 함정 12 — 부분 환불 후 재고 복구 잘못
부분 환불 = 일부 옵션 / 일부 수량. 어떤 재고를 얼마나 복구? **사용자 명시 + 정책 문서화**.

---

## 12. 관련

- [[order-stock]] — 주문 → 결제 직전
- [[../pitfalls/transaction-pitfalls]] (작성 예정) — 외부 IO + 락
- webhook-send (다음) — 우리가 외부에 webhook 보내는 쪽
- [[../../../../security/security|↗ security hub]] — HMAC / vault / PCI-DSS
- [[api-design|↑ api-design hub]]
