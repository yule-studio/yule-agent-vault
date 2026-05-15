---
title: "Load shedding / graceful degradation ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:37:00+09:00
tags: [devops, sre, load-shedding, degradation]
---

# Load shedding / graceful degradation ★

**[[sre|↑ sre]]**

---

## 1. 문제

```
peak traffic / 외부 service fail:
  → 모두 처리 시도 → 시스템 cascade fail → 전부 down
  
대안:
  일부 reject (load shed) 또는 reduced feature (degrade)
  → core service 살리기.

→ "100% service 100% 시간 X" 인정.
```

---

## 2. load shedding 알고리즘 (★)

```
A. random
   1% / 10% reject (random)

B. priority based
   lower priority 먼저 reject

C. queue depth
   queue > threshold → reject

D. CPU based
   CPU > 85% → reject

E. latency based
   p99 > threshold → reject (downstream protection)

F. token bucket / leaky bucket
   rate limit

G. concurrency limit
   semaphore
```

---

## 3. priority class (★)

```
모든 traffic 동등 X. 분류:

P0 (highest):
  - critical user action (결제, 가입)
  - health check
  - admin action

P1:
  - main feature (browse, search)

P2:
  - personalization
  - recommendation
  - analytics

P3 (lowest):
  - bot / crawler
  - background job
  - 외부 webhook
```

```java
@PostMapping("/api/recommend")
public Recommendation recommend(...) {
    if (loadShed.shouldShed(Priority.LOW)) {
        return Recommendation.empty();    // 추천 끔
    }
    return service.recommend(...);
}

@PostMapping("/api/checkout")
public Order checkout(...) {
    if (loadShed.shouldShed(Priority.HIGH)) {
        // P0 도 shed? 거의 끝. 매우 신중.
        throw new ServiceUnavailableException();
    }
    return service.checkout(...);
}
```

---

## 4. graceful degradation 패턴

```
1. read-only mode
   write disable → read 만 OK.
   "잠시 후 다시 시도" 안내.

2. fallback content
   recommendation fail → 인기 상품
   personalization fail → default
   search fail → cached top results

3. queue + delay
   sync API → "We'll process and email you"
   async queue 로

4. cache stale OK
   기본 freshness 1min → 1hour OK

5. external service replace
   payment provider fail → backup

6. feature flag
   non-critical feature off
```

---

## 5. application level 구현 예

```java
@Service
public class LoadSheddingService {
    
    private final AtomicInteger activeRequests = new AtomicInteger();
    private static final int MAX_CONCURRENT = 100;
    
    public boolean shouldAccept(Priority priority) {
        int current = activeRequests.get();
        
        // P0 = 항상 accept (마지막 까지)
        if (priority == Priority.CRITICAL) return true;
        
        // P1 = 90% capacity 까지
        if (priority == Priority.HIGH && current < MAX_CONCURRENT * 0.9) return true;
        
        // P2 = 70%
        if (priority == Priority.MEDIUM && current < MAX_CONCURRENT * 0.7) return true;
        
        // P3 = 50%
        if (priority == Priority.LOW && current < MAX_CONCURRENT * 0.5) return true;
        
        return false;
    }
    
    public <T> T execute(Priority priority, Supplier<T> work) {
        if (!shouldAccept(priority)) {
            throw new LoadShedException();
        }
        activeRequests.incrementAndGet();
        try {
            return work.get();
        } finally {
            activeRequests.decrementAndGet();
        }
    }
}
```

---

## 6. nginx / Envoy 에서

```nginx
# nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

server {
    location /api {
        # P0 = 무제한
        if ($uri = /api/checkout) {
            limit_req zone=api burst=200 nodelay;
        }
        # P3 = 엄격
        if ($uri = /api/analytics) {
            limit_req zone=api burst=5 nodelay;
        }
    }
}
```

```yaml
# Envoy
http_filters:
  - name: envoy.filters.http.local_ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
      stat_prefix: rate_limit
      token_bucket:
        max_tokens: 100
        tokens_per_fill: 50
        fill_interval: 1s
```

---

## 7. adaptive concurrency (★ Netflix)

```
fixed concurrency limit → 시점 별 부적합.
adaptive = 자동 조정:
  - latency 증가 → limit ↓
  - latency 안정 → limit ↑
  - downstream 추적

도구:
  - Netflix Hystrix (legacy, deprecated)
  - Resilience4j AdaptiveBulkhead
  - Envoy adaptive concurrency filter
```

---

## 8. queue overflow

```
SQS / RabbitMQ / Kafka 의 queue overflow:

option A: drop
  oldest 또는 newest

option B: dead-letter queue
  처리 못 한 message 별도 보관

option C: backpressure to producer
  producer 가 stop / slow down

option D: scale consumer
  capacity 추가 (시간 걸림)
```

---

## 9. circuit breaker + load shed 조합

```
external service fail:
  → circuit breaker OPEN
  → fallback (degraded)
  → load 안 들어옴

internal overload:
  → load shedding
  → lower priority reject
  → core 살림

→ 두 mechanism 다 필요.
```

---

## 10. monitoring (★)

```
metric:
  - active concurrent requests
  - shed count by priority
  - response code 429 / 503 rate
  - downstream latency
  - queue depth
  - CPU / memory utilization

alert:
  - shed rate > 5%
  - P0 shed (★ 매우 critical)
  - cascade 의 신호
```

---

## 11. Game Day / chaos

```
load shedding 의 검증:
  - 인위 load (k6)
  - 외부 service mock fail
  - shed count 측정
  - 사용자 경험 측정
  - core service 계속 동작 확인

→ 실제 incident 시 발동되어야.
```

---

## 12. UX 의 graceful degradation

```
사용자 경험:
  - "잠시 시스템이 느려요" 안내
  - progressive disclosure (load 가능한 것만)
  - skeleton UI (loading)
  - 다시 시도 button
  - email 으로 결과 알림 (async)

→ "fail" 대신 "limited" 의 메시지.
```

---

## 13. SLO 와의 관계

```
Error Budget Policy:
  budget 50% 소모 → 새 feature freeze
  
load shedding 도:
  "P3 의 SLO 는 99% — 100% 가 아님"
  "shed 도 정상의 일부"

→ SLO 가 reality. 100% 아님.
```

---

## 14. 함정

1. **P0 만 — 모두 P0** — load shed 의미 X.
2. **load shed 없이 무한 retry** — cascade.
3. **degradation 없음** — "모두 OK 또는 down" 만.
4. **load shed monitoring 안 함** — 발동 모름.
5. **shed 의 UX 부재** — 사용자 그냥 ERROR.
6. **Game Day 안 함** — 실제 incident 첫 시도.
7. **adaptive limit 의 oscillation** — tuning 필요.

---

## 15. 관련

- [[sre|↑ sre]]
- [[../distributed-systems/circuit-breaker|↗ circuit breaker]]
- [[../distributed-systems/backpressure|↗ backpressure]]
- [[slo-management]]
