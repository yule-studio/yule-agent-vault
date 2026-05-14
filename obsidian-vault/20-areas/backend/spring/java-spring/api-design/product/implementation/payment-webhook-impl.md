---
title: "PG webhook 수신 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:28:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - webhook
---

# PG webhook 수신 구현 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. Controller

```java
@RestController
@RequiredArgsConstructor
public class TossWebhookController {

    private final TossSignatureVerifier verifier;
    private final WebhookEventRepository webhooks;
    private final ApplicationEventPublisher events;
    private final ObjectMapper mapper;
    private final IdGenerator ids;
    private final Clock clock;

    @PostMapping("/api/v1/webhooks/toss")
    public ResponseEntity<Void> receive(
            @RequestBody String rawBody,
            @RequestHeader("tosspayments-signature") String signature) {

        // 1. 서명
        if (!verifier.verify(rawBody, signature))
            return ResponseEntity.status(401).build();

        var event = mapper.readValue(rawBody, TossWebhookEvent.class);

        // 2. dedup INSERT (UNIQUE event_id)
        try {
            webhooks.save(new WebhookEvent(
                ids.next(),
                event.eventId(),
                "TOSS",
                event.eventType(),
                signature,
                rawBody,
                false,
                clock.now()));
        } catch (DataIntegrityViolationException e) {
            return ResponseEntity.ok().build();   // already received
        }

        // 3. 비동기 처리 trigger (AFTER_COMMIT)
        events.publishEvent(new PgWebhookReceived(event));

        // 4. 빠른 200 응답
        return ResponseEntity.ok().build();
    }
}
```

---

## 2. Handler (AFTER_COMMIT)

```java
@Component
@RequiredArgsConstructor
public class PgWebhookHandler {

    private final PaymentRepository payments;
    private final WebhookEventRepository webhooks;
    private final Clock clock;

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handle(PgWebhookReceived ev) {
        try {
            var payment = payments.findByPgPaymentKey("TOSS", ev.paymentKey())
                .orElseThrow();

            switch (ev.eventType()) {
                case "payment.approved" -> {
                    if (payment.status() != PaymentStatus.DONE)
                        payment.approve(clock.now());
                }
                case "payment.canceled" -> {
                    if (payment.status() != PaymentStatus.CANCELED)
                        payment.cancel(payment.amount(), "PG_WEBHOOK", null, clock.now());
                }
                case "payment.failed" -> {
                    if (payment.status() != PaymentStatus.FAILED)
                        payment.fail(ev.failureReason(), clock.now());
                }
                default -> log.warn("unknown event: {}", ev.eventType());
            }
            payments.save(payment);
            webhooks.markProcessed(ev.eventId(), clock.now());

        } catch (Exception e) {
            webhooks.markError(ev.eventId(), e.getMessage(), clock.now());
            log.error("webhook handle failed", e);
        }
    }
}
```

---

## 3. 재처리 (admin)

```java
@PostMapping("/api/v1/admin/webhooks/{eventId}/replay")
public void replay(@PathVariable String eventId) {
    var webhook = webhooks.findByEventId(eventId).orElseThrow();
    var event = parse(webhook.rawBody());
    events.publishEvent(new PgWebhookReceived(event));
}
```

→ admin 이 실패한 webhook 수동 replay.

---

## 4. 함정

### 함정 1 — DTO 로 받기 (raw body X)
서명 input 변경.
→ String rawBody.

### 함정 2 — 5xx 응답
PG 가 무한 retry → DDoS.
→ 처리 실패해도 200 + dlq.

### 함정 3 — handler 가 트랜잭션 안
DB 락 / 외부 호출.
→ AFTER_COMMIT.

### 함정 4 — payment 의 status 검증 X
같은 webhook 2번 → 결제 2번 approve.
→ status 검증.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/webhook-strategy]]
- [[../security/webhook-signature]]
- [[../database/webhook-events-table]]
- [[../design-decisions/kafka-event-driven]]
