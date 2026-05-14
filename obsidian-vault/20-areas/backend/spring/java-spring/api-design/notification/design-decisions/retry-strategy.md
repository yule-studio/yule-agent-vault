---
title: "Retry 전략 — exp backoff + max + DLQ"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - retry
---

# Retry 전략 — exp backoff + max + DLQ

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault 정책

- Transient (5xx / timeout / rate) → exp backoff retry max 5회.
- Permanent (invalid token / bad email) → 즉시 DLQ.
- 5회 후 DLQ + admin alert.

```
attempt 1: 즉시
attempt 2: 30s 후
attempt 3: 2 min 후
attempt 4: 10 min 후
attempt 5: 30 min 후
DLQ
```

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 Transient vs Permanent 분류

**왜 필요**
- Permanent 도 retry → 비용 + 시간 낭비.
- 빠른 분류 → 자동 정리 (token cleanup).

**안 하면**
- invalid token 도 5회 retry → 30 분 후 DLQ — token 도 안 청소.
- 비용 5배.

### 2.2 왜 exp backoff (constant 아님)

- 일시 장애는 보통 즉시 복구 — 처음은 30s 빠르게.
- 점점 늘려서 외부 시스템 회복 시간 확보.

### 2.3 왜 max 5회

- 5회 = 약 45 분.
- 그 이상 → 자동 복구 거의 X.
- DLQ + admin = 사람 개입.

**대안**
| 모델 | 적용 |
| --- | --- |
| **5회 exp backoff** ★ (본 vault) | 일반 |
| 3회 fast | latency 우선 |
| 무한 retry (linear) | 위험 |
| Circuit breaker | 채널 전체 장애 시 |

---

## 3. 코드

```java
public class RetryPolicy {
    private static final long[] BACKOFFS = {
        Duration.ofSeconds(30).toMillis(),
        Duration.ofMinutes(2).toMillis(),
        Duration.ofMinutes(10).toMillis(),
        Duration.ofMinutes(30).toMillis()
    };

    public Instant nextAttempt(int attempts, Instant now) {
        if (attempts >= BACKOFFS.length) return null;   // → DLQ
        return now.plus(BACKOFFS[attempts], ChronoUnit.MILLIS);
    }
}
```

```java
public void scheduleRetry(NotificationOutboxRow row, Exception e) {
    if (row.attempts() >= row.maxAttempts()) {
        row.moveToDlq(e.getMessage(), now());
        slack.alert("dlq", row);
    } else {
        var next = retryPolicy.nextAttempt(row.attempts(), now());
        row.recordTransientFailure(e.getMessage(), next);
    }
    repo.save(row);
}
```

---

## 4. Circuit breaker (채널 전체 장애)

```java
// Resilience4j
@Component
public class FcmChannel implements NotificationChannel {

    @CircuitBreaker(name = "fcm", fallbackMethod = "fallback")
    public DeliveryResult send(...) { ... }

    private DeliveryResult fallback(NotificationPayload p, UserDevice d, Throwable e) {
        // 회로 OPEN — FCM 일시 장애 가정
        return DeliveryResult.transient("circuit-open");
    }
}
```

- 1 분 안 50% 실패 → OPEN.
- 30 초 후 HALF_OPEN → 1 요청 test.
- 성공 시 CLOSED.

---

## 5. 함정

### 함정 1 — Transient/Permanent 분류 없음
모든 실패 5회 retry — 비용 ↑↑.

### 함정 2 — Constant backoff
외부 시스템 회복 시간 부족.

### 함정 3 — 무한 retry
DB / 외부 cost 폭주.

### 함정 4 — DLQ 알림 없음
admin 이 모름.

### 함정 5 — Circuit breaker 없음
한 채널 장애 시 모든 worker block.

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[outbox-pattern]]
- [[../implementation/outbox-worker-impl]]
- [[../pitfalls/outbox-pitfalls]]
