---
title: "Webhook 송신 (서명 / 재시도 / DLQ)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - webhook
---

# Webhook 송신 (서명 / 재시도 / DLQ)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | outbox + worker / HMAC 서명 / 재시도 backoff / DLQ |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]]. 수신 측: [[payment-pg#6]] (webhook 수신).

---

## 1. 무엇을 만드는가

우리 서비스 → 외부 시스템 (고객사 / 파트너) 에 이벤트 전송.

```
Event (OrderPaid) ──→ outbox row insert (트랜잭션 안)
                          ↓
                  (별도 worker, 1초 간격 polling)
                          ↓
                      대상 endpoint 마다 HTTP POST
                          + X-Signature: HMAC-SHA-256(body, secret)
                          + X-Idempotency-Key (event ID)
                          ↓
                     200 OK → SENT
                     실패 → retry (exponential backoff)
                     7번 실패 → DEAD_LETTER → 알람
```

### 1.1 사용처

- 주문 / 결제 / 환불 알림을 셀러에게
- 가입 / 탈퇴 이벤트를 마케팅 시스템에
- 재고 변동을 ERP 에
- 사용자 행동 (검색 / 구매) 을 BI 시스템에

---

## 2. 도메인 / DB

```sql
CREATE TABLE webhook_subscriptions (
    id            CHAR(26) PRIMARY KEY,
    subscriber_id CHAR(26) NOT NULL,              -- 셀러 / 파트너 / 시스템 ID
    target_url    VARCHAR(500) NOT NULL,
    event_types   VARCHAR(500) NOT NULL,           -- "ORDER_PAID,ORDER_CANCELED" CSV
    secret        VARCHAR(100) NOT NULL,           -- 서명용 HMAC secret (생성 시 1회 노출)
    active        BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id                CHAR(26) PRIMARY KEY,
    subscription_id   CHAR(26) NOT NULL REFERENCES webhook_subscriptions(id),
    event_id          CHAR(26) NOT NULL,            -- 도메인 event 의 ID (Idempotency-Key)
    event_type        VARCHAR(50) NOT NULL,
    payload           JSONB NOT NULL,
    status            VARCHAR(20) NOT NULL,         -- PENDING / SENT / RETRY / DEAD_LETTER
    attempts          INTEGER NOT NULL DEFAULT 0,
    next_attempt_at   TIMESTAMPTZ NOT NULL,
    last_error        TEXT,
    last_response_status INTEGER,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at           TIMESTAMPTZ
);
CREATE UNIQUE INDEX ux_webhook_deliveries_event ON webhook_deliveries (subscription_id, event_id);
CREATE INDEX ix_webhook_deliveries_due ON webhook_deliveries (status, next_attempt_at)
    WHERE status IN ('PENDING', 'RETRY');
```

`subscription_id + event_id` UNIQUE — 같은 이벤트 한 번만 enqueue (멱등).

---

## 3. Outbox enqueue

```java
// application/webhook/EnqueueWebhookUseCase.java
@Service
@RequiredArgsConstructor
public class EnqueueWebhookUseCase {

    private final WebhookSubscriptionRepository subs;
    private final WebhookDeliveryRepository deliveries;
    private final IdGenerator ids;
    private final Clock clock;
    private final ObjectMapper json;

    /**
     * 같은 트랜잭션에서 호출.
     * 도메인 이벤트 (OrderPaid 등) 의 listener 가 호출.
     */
    @Transactional(propagation = Propagation.MANDATORY)
    public void enqueue(String eventType, String eventId, Object payload) {
        var active = subs.findActiveByEventType(eventType);
        for (var sub : active) {
            try {
                deliveries.save(new WebhookDelivery(
                    new WebhookDeliveryId(ids.next()),
                    sub.id(), eventId, eventType,
                    json.valueToTree(payload),
                    DeliveryStatus.PENDING,
                    0,
                    Instant.now(clock),           // next_attempt_at — 즉시 worker 가 픽업
                    null, null,
                    Instant.now(clock), null
                ));
            } catch (DataIntegrityViolationException dup) {
                // 같은 event_id 가 이미 enqueue 됨 — 무시 (멱등)
            }
        }
    }
}
```

사용:
```java
// OrderPaid 도메인 이벤트 listener
@Component
@RequiredArgsConstructor
public class OrderPaidWebhookListener {
    private final EnqueueWebhookUseCase enqueue;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void on(OrderPaid event) {
        enqueue.enqueue("ORDER_PAID", event.orderId().value(),
            new OrderPaidPayload(event.orderId().value(), event.amount().amountMinorUnit(),
                                 event.occurredAt()));
    }
}
```

→ **`BEFORE_COMMIT`** — outbox INSERT 가 주 트랜잭션과 같이 commit. 트랜잭션 안전.

---

## 4. Worker

```java
// infrastructure/webhook/WebhookDeliveryWorker.java
@Component
@RequiredArgsConstructor
@Slf4j
public class WebhookDeliveryWorker {

    private final WebhookDeliveryRepository deliveries;
    private final WebhookSubscriptionRepository subs;
    private final WebhookHttpClient http;
    private final Clock clock;

    @Scheduled(fixedDelay = 1000)
    @SchedulerLock(name = "webhookDeliveryWorker", lockAtMostFor = "60s", lockAtLeastFor = "5s")
    public void process() {
        var batch = deliveries.findDue(Instant.now(clock), 50);
        for (var d : batch) deliverOne(d.id());
    }

    @Transactional
    public void deliverOne(WebhookDeliveryId deliveryId) {
        var delivery = deliveries.findByIdForUpdate(deliveryId).orElseThrow();
        if (delivery.status() != DeliveryStatus.PENDING && delivery.status() != DeliveryStatus.RETRY)
            return;

        var sub = subs.findById(delivery.subscriptionId()).orElseThrow();
        if (!sub.active()) {
            delivery.markDeadLetter("subscription inactive");
            deliveries.save(delivery);
            return;
        }

        try {
            var payloadStr = json.writeValueAsString(delivery.payload());
            var signature = hmacSha256Hex(payloadStr, sub.secret());

            var response = http.post(sub.targetUrl(), payloadStr, Map.of(
                "Content-Type", "application/json",
                "X-Webhook-Event", delivery.eventType(),
                "X-Webhook-Delivery", delivery.id().value(),
                "X-Webhook-Signature", "sha256=" + signature,
                "X-Idempotency-Key", delivery.eventId(),
                "User-Agent", "Shop-Webhook/1.0"
            ));

            if (response.statusCode() >= 200 && response.statusCode() < 300) {
                delivery.markSent(Instant.now(clock), response.statusCode());
            } else {
                delivery.recordFailure(response.statusCode(),
                    response.body(), computeNextAttempt(delivery.attempts() + 1));
            }
        } catch (Exception e) {
            delivery.recordFailure(null, e.getMessage(),
                computeNextAttempt(delivery.attempts() + 1));
        }
        deliveries.save(delivery);
    }

    /** Exponential backoff: 30s, 5m, 30m, 2h, 6h, 12h, 24h → DEAD_LETTER */
    private Instant computeNextAttempt(int attempt) {
        var seconds = switch (attempt) {
            case 1 -> 30L;
            case 2 -> 300L;
            case 3 -> 1800L;
            case 4 -> 7200L;
            case 5 -> 21600L;
            case 6 -> 43200L;
            case 7 -> 86400L;
            default -> -1L;
        };
        if (seconds < 0) return Instant.MAX;       // 더 이상 시도 X — DEAD_LETTER
        return Instant.now(clock).plusSeconds(seconds);
    }

    private static String hmacSha256Hex(String body, String secret) {
        try {
            var mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(UTF_8), "HmacSHA256"));
            var bytes = mac.doFinal(body.getBytes(UTF_8));
            return HexFormat.of().formatHex(bytes);
        } catch (Exception e) { throw new IllegalStateException(e); }
    }
}
```

```java
// 도메인 단 — recordFailure / markSent / markDeadLetter
public void recordFailure(Integer status, String error, Instant nextAttempt) {
    this.attempts++;
    this.lastResponseStatus = status;
    this.lastError = truncate(error, 1000);
    if (nextAttempt.equals(Instant.MAX)) {
        this.status = DeliveryStatus.DEAD_LETTER;
    } else {
        this.status = DeliveryStatus.RETRY;
        this.nextAttemptAt = nextAttempt;
    }
}
```

---

## 5. HTTP Client (timeout / circuit breaker)

```java
@Component
public class WebhookHttpClient {

    private final HttpClient http = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();

    @CircuitBreaker(name = "webhook", fallbackMethod = "fallback")
    public Response post(String url, String body, Map<String, String> headers) {
        var builder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .timeout(Duration.ofSeconds(10))            // 응답 timeout
            .POST(HttpRequest.BodyPublishers.ofString(body));
        headers.forEach(builder::header);

        try {
            var res = http.send(builder.build(), HttpResponse.BodyHandlers.ofString());
            return new Response(res.statusCode(), res.body());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    private Response fallback(String url, String body, Map<String, String> headers, Throwable t) {
        return new Response(0, "circuit open: " + t.getMessage());
    }

    public record Response(int statusCode, String body) {}
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      webhook:
        failure-rate-threshold: 50
        sliding-window-size: 50
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 5
```

---

## 6. Subscription 관리 endpoint

```java
@RestController
@RequestMapping("/api/v1/webhooks/subscriptions")
@PreAuthorize("hasAnyRole('SELLER', 'ADMIN', 'MASTER')")
@RequiredArgsConstructor
public class WebhookSubscriptionController {

    private final WebhookSubscriptionService service;

    @PostMapping
    public ResponseEntity<CommonResponse<SubscriptionCreated>> create(
        @Valid @RequestBody CreateSubRequest req, Authentication auth
    ) {
        var sub = service.create(currentSubscriberId(auth), req);
        // secret 은 응답에 한 번만 노출
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            new SubscriptionCreated(sub.id(), sub.secret()),
            "구독 생성. secret 은 다시 노출되지 않습니다."));
    }

    @GetMapping
    public ResponseEntity<CommonResponse<List<SubscriptionDto>>> list(Authentication auth) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            service.listByOwner(currentSubscriberId(auth)), "조회"));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<CommonResponse<Void>> delete(@PathVariable String id, Authentication auth) {
        service.delete(id, currentSubscriberId(auth));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "삭제"));
    }

    @PostMapping("/{id}/test")
    public ResponseEntity<CommonResponse<Void>> testSend(@PathVariable String id, Authentication auth) {
        service.sendTest(id, currentSubscriberId(auth));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "테스트 이벤트 발송"));
    }

    @GetMapping("/{id}/deliveries")
    public ResponseEntity<CommonResponse<Page<DeliveryDto>>> deliveryHistory(
        @PathVariable String id,
        @RequestParam(defaultValue = "0") int page,
        Authentication auth
    ) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            service.deliveryHistory(id, currentSubscriberId(auth), page), "조회"));
    }
}
```

→ **테스트 endpoint** + **delivery 히스토리** 가 운영 디버깅에 필수.

---

## 7. Dead Letter 처리

```java
@Scheduled(cron = "0 0 9 * * *")          // 매일 오전 9시
@SchedulerLock(name = "deadLetterReport")
public void reportDeadLetters() {
    var deadLetters = deliveries.findRecentDeadLetters(Duration.ofDays(1));
    if (deadLetters.isEmpty()) return;

    var byEndpoint = deadLetters.stream()
        .collect(Collectors.groupingBy(d -> d.targetUrl(), Collectors.counting()));

    slackAlerter.send("Webhook DLQ alert", byEndpoint.toString());
    // 자주 dead-letter 발생하는 endpoint 는 subscription 비활성 자동 처리 옵션
}

// Admin 화면 — 수동 재시도
@PostMapping("/admin/webhook-deliveries/{id}/retry")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<...> manualRetry(@PathVariable String id) {
    deliveries.findById(id).ifPresent(d -> {
        d.requeue();                       // attempts = 0, status = PENDING, next = now
        deliveries.save(d);
    });
    return ResponseEntity.ok(...);
}
```

---

## 8. 수신자 측 가이드 (우리 사용자에게 보여줄 docs)

- HTTPS only
- 5 분 내 응답 (아니면 timeout → retry)
- HMAC-SHA-256 검증 (`X-Webhook-Signature: sha256=<hex>`)
- `X-Idempotency-Key` 로 중복 처리 차단 (우리가 retry 함)
- 200 응답이면 SENT. 4xx 는 즉시 영구 실패, 5xx 는 retry

```http
POST https://customer.example.com/webhooks/order
Content-Type: application/json
X-Webhook-Event: ORDER_PAID
X-Webhook-Delivery: 01HZWHK...
X-Webhook-Signature: sha256=abc123...
X-Idempotency-Key: 01HZORDER...
User-Agent: Shop-Webhook/1.0

{ "orderId": "...", "amount": 98000, "occurredAt": "2026-05-14T..." }
```

---

## 9. 함정 모음

### 함정 1 — 트랜잭션 안에서 동기 HTTP
주 트랜잭션이 외부 HTTP 응답을 기다림. **outbox + 별도 worker**.

### 함정 2 — 4xx vs 5xx 동일 retry
4xx (badRequest / forbidden) 는 재시도해도 실패. **4xx = 즉시 dead, 5xx = retry**.

### 함정 3 — 무한 재시도
7회 시도 후 dead. 알람 + 수동 검토.

### 함정 4 — 같은 event 가 여러 번 enqueue
이벤트 listener 실행 X 번 = X 개 delivery. **`(subscription_id, event_id) UNIQUE`** + try-catch DataIntegrityViolation.

### 함정 5 — secret 가 코드 / DB plaintext
secret 은 생성 시 1회 노출 + DB 에 hash 저장 옵션 (단 webhook 발송 시 raw 필요 → 암호화 저장).

### 함정 6 — circuit breaker 없음
한 subscriber 가 죽으면 worker 가 그것만 계속 시도 → 다른 잘 동작하는 subscriber 도 느려짐. **endpoint 별 circuit breaker**.

### 함정 7 — payload schema 변경
v1 schema 의 receiver 가 v2 로 받으면 깨짐. **payload 에 `version: 1` 필드** + 신규 필드 추가 (breaking 금지).

### 함정 8 — SSRF
악의적 사용자가 `target_url=http://169.254.169.254/...` (AWS metadata) 등록. **private IP 검증 + 화이트리스트 또는 IP 차단**.

### 함정 9 — payload 너무 큼
1MB+ 이벤트 = 수신자 부담. payload 는 ID + 변경 timestamp 만, 상세는 GET 으로.

### 함정 10 — 발송 시간 → 도착 시간 = 0 기대
HTTP 본질적으로 비동기 + 지연. SLA 명시 (예: 99% 30초 내).

---

## 10. 운영 체크리스트

- [ ] outbox + 별도 worker
- [ ] ShedLock 으로 worker 단일 실행
- [ ] HMAC-SHA-256 서명
- [ ] X-Idempotency-Key (event ID)
- [ ] exponential backoff
- [ ] DLQ + 알람
- [ ] subscriber 별 circuit breaker
- [ ] target_url 의 SSRF 검증 (private IP 차단)
- [ ] secret 가 vault / 암호화
- [ ] subscription 별 delivery 히스토리 admin 화면
- [ ] payload version 명시
- [ ] retry 큐 길이 모니터

---

## 11. 관련

- [[payment-pg]] — 수신 webhook (반대편)
- [[../common/security-config]] — HMAC 검증
- [[distributed-lock]] — ShedLock
- [[../pitfalls/cache-stampede]] (예정) — circuit breaker 패턴
- [[api-design|↑ api-design hub]]
