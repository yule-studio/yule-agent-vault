---
title: "PG webhook 전략 — 비동기 보강 + idempotent + 서명"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - webhook
  - pg
---

# PG webhook 전략 — 비동기 보강 + idempotent + 서명

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> PG 가 보내는 비동기 알림 (결제 승인 / 실패 / 환불 등) 의 처리 전략.
> 본 vault: **/confirm 동기 결과 우선 + webhook 보강** (Toss 표준).

---

## 1. 본 vault 결정

- /confirm 응답이 결제 결과의 **권위 source** — 사용자에게 즉시 보여줌.
- webhook 은 **보강 (idempotent)** — /confirm timeout / 실패 시 cover.
- 모든 webhook: **HMAC-SHA-256 서명 검증** + **dedup 24h** + **AFTER_COMMIT 처리**.

---

## 2. 왜 필요

### 2.1 왜 webhook 이 필수

- /confirm 후 PG 서버 / 카드사 처리에서 추가 상태 변화 발생 (예: 미승인 → 승인 변경).
- 사용자가 PG 결제창을 닫거나 네트워크 끊기면 FE 의 /confirm 호출 X → server 는 모름.
- PG 가 결제 사후 (3일 이내) 위험 거래 판단 → 자동 cancel.

### 2.2 왜 idempotent 필수

- PG 가 동일 event 를 1~5회 재전송 (응답 200 받기 전).
- 같은 event 2번 처리 시 → 결제 2번 / 환불 2번 = 사고.

### 2.3 왜 서명 검증

- webhook URL 이 공개 → 임의의 client 가 fake 결제 알림 전송 가능.
- HMAC = PG 와 서버만 공유하는 secret 으로 서명 검증.

---

## 3. 안 하면 어떤 문제

| 잘못 | 사고 |
| --- | --- |
| webhook 무시 (/confirm 만) | PG 가 사후 cancel 했는데 server 는 모름 → 환불 처리 누락 |
| 서명 검증 X | attacker 가 fake "DONE" 보내서 결제 완료된 척 |
| idempotent X | PG retry 시 결제 2배 |
| 트랜잭션 안에서 처리 | webhook 응답 늦어짐 → PG timeout → retry → idempotent 처리 |
| 응답 200 안 보냄 | PG 가 무한 retry (3일) → DDoS 효과 |
| 순서 의존 | 결제 시도 → 결제 승인 → 환불 의 순서 못 받을 수 있음 (재정렬 X) |

---

## 4. 대안 비교

| 패턴 | 설명 | 장점 | 단점 |
| --- | --- | --- | --- |
| **본 vault: /confirm + webhook 보강** ★ | 동기 + 비동기 | UX 즉시 + 신뢰 ↑ | 2 곳 처리 — 로직 일관성 주의 |
| webhook only | /confirm 안 함 | 단순 | 사용자가 30s 기다림 — UX ↓ |
| /confirm only (webhook 무시) | 단순 | 빠름 | PG 사후 cancel 누락 |
| polling | 서버가 PG 에 주기적 조회 | webhook 의존 X | PG fee + 느림 |
| 이벤트 큐 (SQS) | webhook → SQS → worker | scale + retry 기본 | 인프라 ↑ |

---

## 5. webhook 처리 핵심 (4단계)

```
1. 빠르게 200 응답 (5초 안에) — PG 가 timeout 안 하게
2. 서명 검증 — HMAC
3. dedup — event_id 24h 캐시
4. AFTER_COMMIT 처리 — DB 안 정합성
```

### 5.1 코드

```java
@PostMapping("/webhooks/toss")
public ResponseEntity<Void> tossWebhook(
        @RequestBody String rawBody,
        @RequestHeader("tosspayments-signature") String signature) {

    // 1. 서명
    if (!signatureVerifier.verify(rawBody, signature, tossSecret))
        return ResponseEntity.status(401).build();

    var event = mapper.readValue(rawBody, TossWebhookEvent.class);

    // 2. dedup
    if (webhookEvents.exists(event.eventId()))
        return ResponseEntity.ok().build();          // already processed

    // 3. record (idempotent INSERT)
    try {
        webhookEvents.save(new WebhookEvent(event.eventId(), "TOSS",
            rawBody, now()));
    } catch (DataIntegrityViolationException e) {
        return ResponseEntity.ok().build();          // concurrent dedup
    }

    // 4. AFTER_COMMIT 비동기 처리
    publisher.publishEvent(new PgWebhookReceived(event));

    // 5. 200 즉시
    return ResponseEntity.ok().build();
}
```

```java
@Component
class PgWebhookHandler {

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handle(PgWebhookReceived ev) {
        // payment 조회 / 상태 동기화
        var payment = payments.findByPgPaymentKey(ev.paymentKey()).orElse(null);
        if (payment == null) {
            log.warn("payment not found for webhook: {}", ev.paymentKey());
            return;
        }
        switch (ev.status()) {
            case "DONE" -> payment.approve(now());
            case "FAILED" -> payment.fail(ev.failReason(), now());
            case "CANCELED" -> payment.cancel(payment.amount(), "PG", now());
            default -> log.info("unknown status: {}", ev.status());
        }
        payments.save(payment);
    }
}
```

자세히: [[../implementation/payment-webhook-impl]] · [[../security/webhook-signature]].

---

## 6. 트레이드오프

| 결정 | 본 vault | 대안 | 차이 |
| --- | --- | --- | --- |
| /confirm 우선 | 동기 + UX | webhook only | UX 즉시성 |
| dedup window | 24h | 영구 | 메모리 vs 누락 |
| 서명 algorithm | HMAC-SHA-256 | HMAC-SHA-512 / IP only | 안전성 vs 비용 |
| handler 위치 | TransactionalEventListener | @Async | DB tx 일관성 |

---

## 7. PG 별 webhook 특징

| PG | 서명 헤더 | retry | 순서 |
| --- | --- | --- | --- |
| Toss | `tosspayments-signature` (HMAC) | 5회 (exp backoff) | X (idempotency 필수) |
| KCP | body 안에 서명 + key | 3회 | X |
| Stripe | `Stripe-Signature` (HMAC + timestamp) | 3일 동안 | X |
| 이니시스 | IP 화이트리스트 + 단순 서명 | 미명확 | X |

자세히: [[pg-selection#7]].

---

## 8. 함정

### 함정 1 — 응답 200 안 보냄
PG 가 5xx 받으면 무한 retry → DDoS 효과.
→ 처리 실패해도 일단 200 + 별도 dead-letter row.

### 함정 2 — 트랜잭션 안 처리
DB 락 5분 → PG timeout.
→ AFTER_COMMIT.

### 함정 3 — Idempotency 누락
같은 event 2번 처리 → 결제 / 환불 중복.
→ event_id UNIQUE 인덱스.

### 함정 4 — 서명 검증 안 함
fake "DONE" event 받음.
→ HMAC 의무.

### 함정 5 — 순서 의존
PG retry 시 "FAILED" 가 "DONE" 보다 늦게 도착 가능.
→ event 의 timestamp + 도메인 상태 검증.

### 함정 6 — webhook URL 변경 시 옛 URL 무시
PG 가 옛 URL 에 retry — 누락.
→ PG 의 webhook 설정 변경 + 옛 URL 도 200 응답 (1주).

### 함정 7 — payload 의 amount 신뢰
attacker 가 서명 우회 시도 — amount 변조 가능성.
→ webhook 의 amount 보다 PG 서버 재조회 (security depth).

자세히: [[../pitfalls/payment-pitfalls#webhook]].

---

## 9. 다른 컨텍스트

### 9.1 큰 platform (Stripe Connect)

```yaml
webhook: 자체 큐 (SQS) → fan-out → 여러 consumer
```

### 9.2 multi-vendor / marketplace

```yaml
webhook: per-vendor 분리 + 정산 자동
```

### 9.3 구독 (recurring)

```yaml
webhook events: invoice.created / payment.succeeded / payment.failed (dunning)
```

---

## 10. 관련

- [[design-decisions|↑ hub]]
- [[pg-selection]]
- [[payment-flow]]
- [[../security/webhook-signature]]
- [[../database/webhook-events-table]]
- [[../implementation/payment-webhook-impl]]
- [[../pitfalls/payment-pitfalls]]
