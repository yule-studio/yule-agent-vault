---
title: "Backpressure — overload 시 대응"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:47:00+09:00
tags: [devops, distributed-systems, backpressure, resilience]
---

# Backpressure — overload 시 대응

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 무엇

```
upstream traffic > downstream 능력:
  
A: 1000 req/s
B: 100 req/s capacity (10x 부족)

옵션:
  - 모두 받기 → B 가 OOM / lock-up (★ 나쁨)
  - 일부 reject → graceful (★ 권장)
  - upstream 으로 slowdown 신호 → flow control
```

→ "받은 모든 것 처리 시도" 가 cascade failure 의 원인.

---

## 2. load shedding (★ 가장 흔한)

```
overload 감지 → 일부 요청 거부 (503).

방법:
  1. random shed (1% reject)
  2. priority shed (lower priority 먼저)
  3. queue depth 기반 (queue 80% 차면 reject)
  4. CPU 기반 (80% 넘으면)
  5. latency 기반 (p99 > 500ms 면)
```

```java
@RestController
public class ApiController {

    @GetMapping("/api")
    public Response handle() {
        if (getCpuUsage() > 0.85) {
            throw new ServiceUnavailableException("overloaded");
        }
        return process();
    }
}
```

---

## 3. concurrency limit (★)

```
한 service 의 동시 처리 max.

방법:
  - semaphore
  - thread pool
  - bulkhead (Resilience4j)
  - rate limiter

server config:
  Tomcat: max-connections + max-threads
  Nginx: worker_connections
```

```java
Semaphore limit = new Semaphore(100);

public Response handle() {
    if (!limit.tryAcquire(100, MILLISECONDS)) {
        throw new ServiceUnavailableException();
    }
    try {
        return process();
    } finally {
        limit.release();
    }
}
```

---

## 4. queue + backpressure (★)

```
producer → queue → consumer

queue 너무 큼:
  - memory 폭주
  - latency ↑ (FIFO 늦음)
  - eventual fail

backpressure:
  bounded queue (max size)
  queue full → producer block 또는 reject
```

```java
// bounded queue
BlockingQueue<Job> queue = new LinkedBlockingQueue<>(1000);

// producer
boolean offered = queue.offer(job, 100, MILLISECONDS);
if (!offered) {
    throw new BackpressureException("queue full");
}

// consumer
while (true) {
    Job job = queue.take();
    process(job);
}
```

---

## 5. Reactive Streams (★ 표준)

```java
// Reactor (Spring WebFlux)
Flux<String> source = Flux.range(1, 1_000_000)
    .map(i -> "item-" + i)
    .onBackpressureBuffer(100)            // queue 100 까지
    // 또는
    .onBackpressureDrop(dropped -> log.warn("dropped: {}", dropped))
    .onBackpressureLatest();              // 가장 최신 만 keep

Subscriber<String> consumer = ...;
consumer.request(10);   // 10 만 요청
```

→ subscriber 가 demand 조절. publisher 는 demand 만큼만.

---

## 6. Kafka 의 backpressure

```
producer:
  buffer.memory: 32MB (default)
  → buffer 차면 block (또는 throw)

consumer:
  max.poll.records: 500
  fetch.max.bytes: 50MB
  → 한 번에 처리 가능 만큼만 fetch

consumer lag 모니터링:
  lag > threshold → alert
  scale consumer count
```

---

## 7. rate limiting (★)

```
client / API 단:
  per-IP / per-API key / per-user
  
algorithm:
  - token bucket
  - leaky bucket
  - sliding window

도구:
  - Resilience4j RateLimiter
  - Bucket4j (Java)
  - Redis rate limiter
  - nginx limit_req
  - API Gateway
```

```java
RateLimiterConfig config = RateLimiterConfig.custom()
    .limitForPeriod(100)                // 100 / period
    .limitRefreshPeriod(Duration.ofSeconds(1))
    .timeoutDuration(Duration.ofMillis(100))
    .build();
RateLimiter limiter = RateLimiter.of("api", config);

// 사용
limiter.executeSupplier(() -> process());
```

---

## 8. SQS / 비동기 큐

```
synchronous → asynchronous 전환:

client → enqueue (SQS / Kafka) → worker process

장점:
  - immediate 응답 (queued)
  - worker 가 자신 속도로
  - producer 부담 ↓
  - spike 흡수

단점:
  - eventual processing
  - error handling 복잡 (dead letter queue)
```

---

## 9. WebFlux / reactive Spring

```java
@RestController
public class ApiController {

    @GetMapping("/api")
    public Mono<Response> handle(@RequestBody Request req) {
        return client.callAsync(req)              // non-blocking
                     .timeout(Duration.ofSeconds(3))
                     .onErrorResume(this::fallback);
    }
}
```

→ thread 가 wait 안 함. 같은 thread 가 N concurrent handle.

→ 단점: debug 어려움, blocking lib 와 호환 X.

---

## 10. 흔한 시나리오

### A. traffic spike

```
일반 100 RPS → 갑자기 5000 RPS

대응:
  1. autoscaling 빠름 (Karpenter / KEDA)
  2. CDN cache (정적) 흡수
  3. queue 로 async (비-realtime)
  4. circuit break for non-critical
  5. graceful degradation (recommendation 끔)
  6. load shed (low priority 먼저)
```

### B. downstream slow

```
DB 가 평소 5ms → 갑자기 500ms

대응:
  1. circuit break (DB 호출)
  2. cache 적극
  3. timeout (3s)
  4. read replica scale-out
  5. connection pool 안 늘림 (DB 더 부담)
```

### C. cold start

```
서버 재시작 → cache miss / JIT not warm → 모든 request 느림

대응:
  1. readiness probe — traffic 받기 전 warm-up
  2. pre-loading (cache warm)
  3. blue-green (warm 후 cutover)
  4. canary (1% 부터)
```

---

## 11. 더 상위 패턴

```
1. flow control (TCP 처럼)
   sender 가 receiver 의 window 만큼만 send

2. push → pull
   "보낼게" → "필요할 때 요청"

3. token-based
   call 마다 token 필요, 한도 내에서만

4. priority class
   high / medium / low → high 만 처리 시
```

---

## 12. 모니터링

```
metric:
  - queue depth
  - reject / drop count
  - latency p95/p99
  - active connections
  - thread / pool 사용
  - circuit state

alert:
  - reject rate > 1%
  - queue full
  - latency spike
  - downstream timeout 증가
```

---

## 13. 함정

1. **무제한 queue** — OOM.
2. **block 만, reject 안 함** — upstream 영향 (cascade).
3. **rate limit per service total** — 한 user 폭주 가능. per-user.
4. **load shed 의 fairness 없음** — 한 user 만 reject.
5. **autoscale 이 backpressure 대체** — cold start / 외부 dep limit.
6. **모니터링 없음** — overload 인지 안 됨.
7. **reactive 만, blocking lib 사용** — thread block.

---

## 14. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[circuit-breaker]]
- [[../performance/capacity-planning|↗ capacity]]
- [[../sre/incident-response|↗ incident]]
