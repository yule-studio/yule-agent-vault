---
title: "Circuit breaker — fail-fast / fallback"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:45:00+09:00
tags: [devops, distributed-systems, circuit-breaker, resilience]
---

# Circuit breaker — fail-fast / fallback

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 문제

```
service A → service B
B 가 down 또는 매우 느림.

A 가 계속 호출:
  - thread / connection 가 wait 에 쌓임
  - A 의 자원 고갈
  - cascade failure → A 도 down
  - 사용자도 wait
```

→ B 의 문제가 A 로 전파.

---

## 2. 해결: circuit breaker

```
3 state:

CLOSED (정상):
  call → B 로 전달
  
OPEN (B fail 감지):
  call → 즉시 fail (B 안 호출, fast fail)
  → A 자원 보호
  
HALF_OPEN (회복 시도):
  일부 call → B 시도
  성공 → CLOSED
  실패 → OPEN
```

```
state 전환:

CLOSED ─── error rate > 50% ──→ OPEN
                                  ↓ (30s wait)
                              HALF_OPEN
                              ↙       ↘
                          1 success    1 fail
                              ↓           ↓
                          CLOSED       OPEN
```

---

## 3. Resilience4j (★ Java)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slidingWindowSize(20)               // 최근 20 call
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .failureRateThreshold(50)            // 50% fail → OPEN
    .slowCallRateThreshold(80)           // 80% slow → OPEN
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(5)
    .recordExceptions(IOException.class, TimeoutException.class)
    .ignoreExceptions(ValidationException.class)   // 4xx 무시
    .build();

CircuitBreaker breaker = CircuitBreaker.of("payment-service", config);

// 사용
String result = breaker.executeCallable(() -> paymentClient.charge(req));

// fallback
String result = Decorators.ofCallable(() -> paymentClient.charge(req))
    .withCircuitBreaker(breaker)
    .withFallback(throwable -> "fallback response")
    .decorate()
    .call();
```

---

## 4. Spring Cloud Circuit Breaker

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "payment", fallbackMethod = "fallback")
    public Payment charge(ChargeRequest req) {
        return paymentClient.charge(req);
    }
    
    public Payment fallback(ChargeRequest req, Throwable t) {
        log.warn("Payment 우회. 큐에 추가.");
        queue.send("payment-retry", req);
        return Payment.pending();
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        failureRateThreshold: 50
        slidingWindowSize: 20
        waitDurationInOpenState: 30s
```

---

## 5. metric / monitoring

```promql
# Prometheus
resilience4j_circuitbreaker_state{name="payment", state="open"}
resilience4j_circuitbreaker_failure_rate{name="payment"}
resilience4j_circuitbreaker_calls{name="payment", kind="failed"}
```

```
dashboard:
  state (closed/open/half-open) per service
  failure rate trend
  call count + error count
  slow call %
  
alert:
  OPEN state >= 1min → notify
```

---

## 6. fallback 전략

```
1. cached / default response
   "추천 list 못 가져왔으니 인기 상품 보여줌"

2. degraded service
   "결제 못 함 → 큐에 넣고 나중 처리"

3. fast fail with clear error
   "외부 service 일시 불가. 잠시 후 다시."

4. graceful degradation
   "personalization 끔, 기본 content"

5. async / queue
   "Kafka 에 publish, eventual 처리"
```

→ fallback 도 fail 할 수 있음 (nested).

---

## 7. retry + circuit breaker (★)

```java
// 순서: retry 안에 circuit breaker
// 잘못된 순서: circuit breaker 가 retry 횟수만 봄

// 권장:
Retry retry = Retry.of("payment", RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofSeconds(1))
    .exponentialBackoff(1, 2)
    .retryExceptions(IOException.class)
    .build());

CircuitBreaker breaker = CircuitBreaker.of("payment", ...);

// retry inside breaker
Supplier<String> decorated = CircuitBreaker.decorateSupplier(breaker,
    Retry.decorateSupplier(retry, () -> client.call())
);
```

→ retry 가 먼저 (network 일시 hiccup), 그 다음 breaker (실제 down).

---

## 8. timeout (★ 같이 사용)

```java
TimeLimiterConfig timeoutConfig = TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofSeconds(3))
    .build();

CompletableFuture<String> future = Decorators
    .ofCompletionStage(() -> CompletableFuture.supplyAsync(() -> client.call()))
    .withCircuitBreaker(breaker)
    .withTimeLimiter(TimeLimiter.of(timeoutConfig))
    .get()
    .toCompletableFuture();
```

→ timeout 없는 circuit breaker = 의미 ↓.

---

## 9. bulkhead (격리)

```
한 service 의 thread pool 격리.
B 가 느리면 B 호출 thread 만 사용 (전체 thread pool 보호).

Resilience4j Bulkhead:
  semaphore (counter)
  thread pool (별도 executor)
```

```java
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(10)              // 동시 10 만
    .maxWaitDuration(Duration.ofSeconds(1))
    .build();
Bulkhead bulkhead = Bulkhead.of("payment", config);

bulkhead.executeSupplier(() -> client.call());
```

→ 잘못된 한 service 의 thread block 으로 전체 app down 방지.

---

## 10. service mesh 에서

```
Istio / Linkerd 가 자동 circuit breaker:

# DestinationRule
trafficPolicy:
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionTime: 60s
    maxEjectionPercent: 50
```

→ application code 변경 없이 mesh 가 자동.

---

## 11. 패턴 조합 (★ 시니어)

```
client → 
  Timeout (3s)
    ↓
  Retry (3 attempts, exponential backoff)
    ↓
  Circuit Breaker (50% fail → OPEN)
    ↓
  Bulkhead (10 concurrent max)
    ↓
  External Service

Fallback:
  - cache / default
  - queue
  - graceful degradation
```

---

## 12. 실전 예 (★)

```java
@Service
public class ProductService {

    @CircuitBreaker(name = "recommendation", fallbackMethod = "recommendFallback")
    @TimeLimiter(name = "recommendation")
    @Retry(name = "recommendation")
    @Bulkhead(name = "recommendation")
    public CompletableFuture<List<Product>> getRecommendations(Long userId) {
        return recommendationClient.getAsync(userId);
    }
    
    public CompletableFuture<List<Product>> recommendFallback(Long userId, Throwable t) {
        log.warn("recommendation 우회: {}", t.getMessage());
        return CompletableFuture.completedFuture(
            popularProductService.getTopN(10)   // 인기 상품으로 대체
        );
    }
}
```

---

## 13. 함정

1. **circuit 너무 sensitive** — 정상 spike 도 OPEN.
2. **timeout 없음** — request 무한 wait.
3. **fallback 도 fail** — nested fallback 필요.
4. **retry without backoff** — 폭주 가속.
5. **retry idempotent 안 함** — 중복 처리.
6. **monitoring 없음** — circuit 행동 모름.
7. **bulkhead pool 너무 작음** — 정상 traffic 도 reject.

---

## 14. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[backpressure]]
- [[idempotency]]
- [[../networking-ops/service-mesh|↗ service mesh]]
