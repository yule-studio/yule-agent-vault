---
title: "Idempotency-Key — 결제 중복 방어"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:01:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - idempotency
---

# Idempotency-Key — 결제 중복 방어 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

---

## 1. 본 vault 표준

- `Idempotency-Key` 헤더 (Stripe / Toss 표준).
- 같은 key 24h 안에 재호출 시 → 동일 응답.
- 적용: `/payments/confirm` / `/payments/{id}/cancel`.

---

## 2. 왜

- 사용자 더블 클릭 / 네트워크 retry → 같은 결제 2번.
- PG /confirm 응답 timeout 시 클라이언트가 재호출.
- → 멱등 = 1번만 수행.

---

## 3. 구현

```java
@Service
@RequiredArgsConstructor
public class IdempotencyStore {

    private final StringRedisTemplate redis;
    private final ObjectMapper mapper;

    public <T> T execute(String key, Supplier<T> action) {
        var cacheKey = "idempotency:" + key;
        var cached = redis.opsForValue().get(cacheKey);

        if (cached != null) {
            return parse(cached);    // 동일 응답
        }

        // SETNX (lock)
        var locked = redis.opsForValue().setIfAbsent(cacheKey + ":lock", "1", Duration.ofSeconds(30));
        if (!Boolean.TRUE.equals(locked)) {
            throw new ConcurrentRequestException();   // 다른 worker 가 처리 중
        }

        try {
            var result = action.get();
            redis.opsForValue().set(cacheKey,
                mapper.writeValueAsString(result),
                Duration.ofHours(24));
            return result;
        } finally {
            redis.delete(cacheKey + ":lock");
        }
    }
}
```

```java
@PostMapping("/payments/confirm")
public ResponseEntity<PaymentResult> confirm(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentConfirmRequest req) {

    var result = store.execute(idempotencyKey, () ->
        paymentService.confirm(idempotencyKey, req));

    return ResponseEntity.ok(result);
}
```

---

## 4. DB 보조 (Redis fallback)

```sql
ALTER TABLE payments ADD COLUMN idempotency_key VARCHAR(100);
CREATE UNIQUE INDEX ux_payments_idempotency
    ON payments (idempotency_key) WHERE idempotency_key IS NOT NULL;
```

→ Redis 장애 시 DB 의 UNIQUE 가 2차 방어.

---

## 5. 함정

### 함정 1 — Idempotency-Key 헤더 없음
사용자 결제 2번.
→ FE 가 UUID 자동 생성 + 헤더 추가.

### 함정 2 — Redis only (DB X)
Redis 장애 시 멱등 X.
→ DB UNIQUE 보조.

### 함정 3 — TTL 너무 짧음 (1시간)
24h 후 같은 key 의 retry → 결제 2번.
→ 24h ~ 7d.

### 함정 4 — 다른 endpoint 의 key 같음
/confirm 의 key 가 /cancel 에서 hit.
→ key prefix 분리 (`idempotency:confirm:` vs `idempotency:cancel:`).

---

## 6. 관련

- [[security|↑ hub]]
- [[../design-decisions/payment-flow]]
- [[../database/payments-table]]
- [[../implementation/payment-confirm-impl]]
