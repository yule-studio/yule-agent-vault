---
title: "분산 락 (Redisson) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - distributed-lock
  - redisson
---

# 분산 락 (Redisson) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Redisson RLock / TTL / fencing / Spring AOP wrap |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]]. 관련 깊이: [[order-stock#3]] (재고 차감의 분산 락 사용 예).

---

## 1. 분산 락 vs JVM 락 vs DB 락

| | 단일 JVM | 분산 (Redis) | DB (pessimistic) |
| --- | --- | --- | --- |
| 적용 범위 | 한 인스턴스 | 클러스터 전체 | 같은 DB row |
| latency | ns | μs~ms | ms |
| 안정성 | 강함 | TTL 의 단점 (§7) | 가장 강함 |
| 데드락 | JVM 종료로 해결 | TTL 으로 해결 | DB 가 감지 |
| 부하 | JVM | Redis | DB |

→ **다중 인스턴스 + 같은 자원 보호** = 분산 락. **단일 DB row + 짧은 critical section** = 비관 락.

본 레시피는 **Redisson** (가장 널리 쓰임).

---

## 2. 사용 시나리오

| 시나리오 | Lock key | TTL |
| --- | --- | --- |
| 한정수량 재고 차감 | `lock:option:{optionId}` | 3s |
| 사용자 결제 동시 시도 | `lock:user-payment:{userId}` | 10s |
| `@Scheduled` job 다중 인스턴스 | `lock:job:{name}` | job 예상 시간 + 1.5× |
| 외부 API 한 번에 1 호출 | `lock:external-api:{endpoint}` | 5s |
| 캐시 갱신 (stampede 방지) | `lock:cache-refresh:{key}` | 1~3s |
| 쿠폰 first-come-first-serve | `lock:coupon:{couponId}` | 5s |

---

## 3. Redisson 의존성 (이미 설정됨)

```kotlin
implementation("org.redisson:redisson-spring-boot-starter:3.27.2")
```

```yaml
spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    timeout: 3000
```

자동으로 `RedissonClient` Bean 생성됨.

---

## 4. 기본 사용 — `tryLock(wait, lease, unit)`

```java
@Service
@RequiredArgsConstructor
public class StockService {

    private final RedissonClient redisson;
    private final ProductOptionRepository options;
    private final TransactionTemplate tx;

    public void decreaseStock(ProductOptionId optionId, int qty) {
        RLock lock = redisson.getLock("lock:option:" + optionId.value());
        boolean acquired = false;
        try {
            acquired = lock.tryLock(3, 5, TimeUnit.SECONDS);
            // waitTime=3s (락 대기), leaseTime=5s (락 보유 후 자동 해제)
            if (!acquired) {
                throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                    "재고 차감 락 획득 실패 — 잠시 후 다시 시도");
            }

            tx.executeWithoutResult(s -> {
                var opt = options.findById(optionId).orElseThrow();
                opt.decreaseStock(qty);
                options.save(opt);
            });
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException(ResponseCode.INTERNAL_SERVER_ERROR, "락 대기 중 인터럽트");
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

> **핵심**: `try-finally` 로 unlock. `isHeldByCurrentThread()` 체크 — TTL 만료 후엔 다른 스레드가 잡았을 수도.

---

## 5. AOP 식 — `@DistributedLock` annotation

매번 try-finally 반복은 피로. AOP 로 wrap.

```java
// presentation/annotation/DistributedLock.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    /** SpEL — e.g. "'option:' + #optionId.value()" */
    String key();
    long waitSeconds() default 3;
    long leaseSeconds() default 5;
    boolean throwOnFail() default true;
}
```

```java
// presentation/aspect/DistributedLockAspect.java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class DistributedLockAspect {

    private final RedissonClient redisson;
    private final SpelExpressionParser parser = new SpelExpressionParser();
    private final ParameterNameDiscoverer paramDiscoverer = new DefaultParameterNameDiscoverer();

    @Around("@annotation(distLock)")
    public Object around(ProceedingJoinPoint pjp, DistributedLock distLock) throws Throwable {
        var sig = (MethodSignature) pjp.getSignature();
        var key = "lock:" + resolveKey(distLock.key(), sig.getMethod(), pjp.getArgs());
        var rlock = redisson.getLock(key);
        boolean acquired = false;
        try {
            acquired = rlock.tryLock(distLock.waitSeconds(), distLock.leaseSeconds(), TimeUnit.SECONDS);
            if (!acquired) {
                log.warn("distributed lock failed to acquire: {}", key);
                if (distLock.throwOnFail()) {
                    throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                        "락 획득 실패: " + key);
                }
                return null;
            }
            return pjp.proceed();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException(ResponseCode.INTERNAL_SERVER_ERROR, "락 인터럽트");
        } finally {
            if (acquired && rlock.isHeldByCurrentThread()) {
                rlock.unlock();
            }
        }
    }

    private String resolveKey(String spel, Method method, Object[] args) {
        var paramNames = paramDiscoverer.getParameterNames(method);
        var ctx = new StandardEvaluationContext();
        if (paramNames != null) {
            for (int i = 0; i < paramNames.length; i++) {
                ctx.setVariable(paramNames[i], args[i]);
            }
        }
        return parser.parseExpression(spel).getValue(ctx, String.class);
    }
}
```

사용:

```java
@Service
public class StockService {

    @DistributedLock(key = "'option:' + #optionId.value()", waitSeconds = 3, leaseSeconds = 5)
    @Transactional
    public void decreaseStock(ProductOptionId optionId, int qty) {
        var opt = optionRepo.findById(optionId).orElseThrow();
        opt.decreaseStock(qty);
        optionRepo.save(opt);
    }
}
```

> **함정 (트랜잭션 + 락 순서)**: `@DistributedLock` 안에 `@Transactional` 이 있으면 — **락 획득 → 트랜잭션 시작 → DB → 커밋 → 락 해제** 순서. AOP order 가 락 > 트랜잭션 가 되도록 `@Order(Ordered.HIGHEST_PRECEDENCE)` 또는 명시 (락 aspect 가 더 outer).

---

## 6. Fencing Token — Redlock 의 한계 해결

Martin Kleppmann 의 비판: TTL 만료 후 다른 노드가 락 획득 → 옛 노드가 작업 마치고 commit → 두 개 동시.

**해결 — fencing token**: 락 획득마다 단조 증가하는 token. 자원 변경 시 token 도 함께 저장 → 작아진 token 거절.

```java
@Service
public class FencedStockService {

    private final RedissonClient redisson;

    public void decreaseStock(ProductOptionId optionId, int qty) {
        var lock = redisson.getLock("lock:option:" + optionId.value());
        if (!lock.tryLock(3, 5, TimeUnit.SECONDS)) {
            throw new BusinessException(...);
        }
        try {
            // Redisson 은 fencing token 직접 제공 X
            // → 별도 AtomicLong (Redis INCR) 으로 token 생성
            long token = redisson.getAtomicLong("token:option:" + optionId.value()).incrementAndGet();

            // DB 작업 시 token 도 함께 갱신
            var opt = optionRepo.findById(optionId).orElseThrow();
            if (opt.getLastFenceToken() >= token) {
                throw new BusinessException("stale fence token");
            }
            opt.decreaseStockAndUpdateFence(qty, token);
            optionRepo.save(opt);
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }
    }
}
```

→ TTL 만료 시나리오에서도 옛 노드의 commit 이 거절됨.

> 실전에선 대부분 fencing token 까지는 불필요 (TTL 충분히 길게 + 트랜잭션 짧게 = 99% OK). 절대 안전성 필요 시 도입.

---

## 7. TTL 의 함정 — "락 hold time < TTL"

```
TTL = 5s, 작업 = 4s          → OK
TTL = 5s, 작업 = 6s 지연      → TTL 만료, 다른 노드가 락 획득, 옛 노드 commit = 동시
```

**규칙**:
1. **TTL = 작업 예상 시간 + 안전 마진 (3~5x)**
2. **외부 IO 락 안에 두지 말 것** — 예측 불가
3. 트랜잭션 짧게, lock 짧게

```java
@DistributedLock(key = "...", leaseSeconds = 5)
public void decreaseStock(...) {
    // ❌ 락 안에 외부 API 호출 X
    // pgClient.confirm(...)            // 5초 timeout → TTL 위험

    // ✅ DB 만
    opt.decreaseStock(qty);
    optionRepo.save(opt);
}
```

---

## 8. `@Scheduled` 분산 — ShedLock 가 더 적합

`@Scheduled` job 의 다중 인스턴스 동시 실행 방지엔 **ShedLock** (Redisson 기반 가능):

```kotlin
implementation("net.javacrumbs.shedlock:shedlock-spring:5.13.0")
implementation("net.javacrumbs.shedlock:shedlock-provider-redis-spring:5.13.0")
```

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT10M")
public class SchedulerConfig {
    @Bean
    public LockProvider lockProvider(RedisConnectionFactory cf) {
        return new RedisLockProvider(cf);
    }
}

@Component
public class ExpireOrdersJob {
    @Scheduled(fixedDelay = 60_000)
    @SchedulerLock(name = "expireOrders", lockAtLeastFor = "30s", lockAtMostFor = "5m")
    public void run() {
        // ...
    }
}
```

→ 인스턴스 N개 중 한 곳만 실행. 다른 곳은 같은 분에 무시.

본 레시피의 `@DistributedLock` 은 **요청 흐름의 critical section** 용. 스케줄 job 은 ShedLock.

---

## 9. 테스트

```java
@Test
void concurrent_stock_decrease_does_not_oversell() throws Exception {
    var option = optionFixture.createWithStock(50);
    var users = userFixture.createMany(100);

    var executor = Executors.newFixedThreadPool(20);
    var successes = new AtomicInteger();
    var latch = new CountDownLatch(100);

    for (var u : users) {
        executor.submit(() -> {
            try {
                stockService.decreaseStock(option.id(), 1);
                successes.incrementAndGet();
            } catch (Exception ignored) {}
            finally { latch.countDown(); }
        });
    }
    latch.await(30, TimeUnit.SECONDS);
    executor.shutdown();

    assertThat(successes.get()).isEqualTo(50);
    assertThat(optionFixture.reload(option.id()).getStock()).isEqualTo(0);
}
```

---

## 10. 함정 모음

### 함정 1 — finally 에서 unconditional unlock
TTL 만료 후 다른 노드가 락 획득 → 내가 unlock = 그 락 풀림. `isHeldByCurrentThread()` 체크 필수.

### 함정 2 — TTL 너무 짧음
작업이 더 오래 = 옛 락 만료 후 새 락 = 동시 실행. **3~5배 안전 마진**.

### 함정 3 — TTL 너무 김
다른 노드 영원히 대기. 5~10s 권장.

### 함정 4 — 외부 IO 락 안에서
SMTP / PG / S3 등 예측 불가. **락 밖**.

### 함정 5 — 같은 키 다른 작업
공유 락 = 무관 작업끼리 막힘. **세분화** (`lock:order-confirm:{orderId}` 등).

### 함정 6 — 락 wait 무한
`lock(0, 5, SECONDS)` waitTime=0 = 무한 대기. 명시적 timeout.

### 함정 7 — Redis 단일 인스턴스 SPOF
Redis 죽으면 lock 동작 X. **Redis Sentinel / Cluster** + fail-open 정책.

### 함정 8 — Redlock 의 정확성
다중 Redis 마스터의 Redlock 알고리즘 — Kleppmann 의 비판 (clock drift, network 등). 일반 SaaS 엔 단일 Redis + 짧은 TTL + fencing 으로 충분.

### 함정 9 — `@Transactional` 안에서 락 획득
트랜잭션이 락보다 먼저 시작 = 락 hold 시간 ↑. **락 먼저**, 그 안에 트랜잭션 시작.

### 함정 10 — lock key 가 너무 일반적
`lock:global` = 시스템 전체 직렬화 = 처리량 절벽. **항상 자원 ID 포함**.

---

## 11. 운영 체크리스트

- [ ] `isHeldByCurrentThread()` 체크 후 unlock
- [ ] TTL 이 작업 예상 시간 × 3~5
- [ ] 락 안에 외부 IO X
- [ ] AOP order — `@DistributedLock` > `@Transactional`
- [ ] Redis 모니터링 — 락 wait 시간 / 충돌율
- [ ] Redis Sentinel / Cluster 구성 (단일 SPOF X)
- [ ] 정기 락 분포 분석 (어느 키 가장 자주 충돌)
- [ ] `@Scheduled` 는 ShedLock 사용
- [ ] fencing token — 절대 안전성 필요한 도메인만

---

## 12. 관련

- [[order-stock]] — 재고 차감 적용 예
- [[payment-pg]] — 결제 멱등성
- [[cache-redis]] — cache stampede 방지에 락 활용
- [[../pitfalls/concurrency-pitfalls]] (예정)
- [[api-design|↑ api-design hub]]
