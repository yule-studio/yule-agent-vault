---
title: "Rate Limiting (Bucket4j + Redis) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - rate-limit
  - bucket4j
---

# Rate Limiting (Bucket4j + Redis) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | token bucket / IP·user·endpoint 차원 / 429 응답 |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가

특정 endpoint / user / IP 의 호출 빈도 제한. 초과 시 `429 Too Many Requests` + `Retry-After` 헤더.

### 1.1 보호 대상 (대표)

| Endpoint | Limit | 차원 |
| --- | --- | --- |
| `POST /auth/login` | 5회/분 | IP + email |
| `POST /auth/signup` | 3회/시간 | IP |
| `POST /auth/password-reset/request` | 3회/시간 | email |
| `POST /auth/verify/email/request` | 3회/시간 | user |
| `POST /messages/sms/verify` | 5회/시간 | tel + IP |
| `POST /payments/confirm` | 10회/분 | user |
| 일반 GET | 60회/분 | user (옵션) |
| Search | 30회/분 | IP |

### 1.2 알고리즘

| 알고리즘 | 설명 | 본 레시피 |
| --- | --- | --- |
| **Token Bucket** | 일정 속도로 토큰 충전, 요청마다 소비 | ✅ (Bucket4j) |
| Leaky Bucket | 큐 + 일정 속도 처리 | 메시지 큐 식 |
| Fixed Window | "1분에 N회" 계산 — 경계 burst 문제 | 단순 |
| Sliding Window | 마지막 N초 평균 | 정확 / 비용 ↑ |

**Token Bucket** 이 burst 허용 + 평균 제한 — 가장 표준.

---

## 2. 의존성

```kotlin
// build.gradle.kts
implementation("com.bucket4j:bucket4j-redis:8.10.1")
implementation("com.bucket4j:bucket4j-core:8.10.1")
implementation("org.springframework.boot:spring-boot-starter-data-redis")
```

---

## 3. Rate Limit Service (port + adapter)

```java
// domain/ratelimit/RateLimiter.java
public interface RateLimiter {
    /** 토큰 1개 소비 시도. true = 성공, false = 거절. */
    ConsumptionResult tryConsume(String key, RateLimitPolicy policy);

    record ConsumptionResult(boolean allowed, long remainingTokens, Duration retryAfter) {}
}

public record RateLimitPolicy(long capacity, Duration refillPeriod) {
    public static RateLimitPolicy of(long capacity, Duration refill) {
        return new RateLimitPolicy(capacity, refill);
    }
}
```

### 3.1 Bucket4j + Redis adapter

```java
// infrastructure/ratelimit/Bucket4jRedisRateLimiter.java
@Component
@RequiredArgsConstructor
public class Bucket4jRedisRateLimiter implements RateLimiter {

    private final RedissonClient redisson;

    @Override
    public ConsumptionResult tryConsume(String key, RateLimitPolicy policy) {
        var proxyManager = RedissonBasedProxyManager.builderFor(redisson)
            .withExpirationStrategy(ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(Duration.ofHours(1)))
            .build();

        var config = BucketConfiguration.builder()
            .addLimit(limit -> limit
                .capacity(policy.capacity())
                .refillIntervally(policy.capacity(), policy.refillPeriod()))
            .build();

        var bucket = proxyManager.builder().build(key, () -> config);
        var probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            return new ConsumptionResult(true, probe.getRemainingTokens(), null);
        }
        return new ConsumptionResult(false, 0,
            Duration.ofNanos(probe.getNanosToWaitForRefill()));
    }
}
```

> **함정**: Bucket4j 의 API 가 버전에 따라 변경. 8.x 부터 lambda + builder 형태. 정확한 메서드명은 docs 확인.

### 3.2 Java HashMap 식 (in-memory) — 단일 인스턴스 / dev 용

```java
@Profile("test | local")
@Component
public class LocalRateLimiter implements RateLimiter {
    // ConcurrentHashMap + AtomicLong — 단일 JVM 만. 운영엔 X.
}
```

---

## 4. 적용 — 3가지 방법

### 4.1 Filter (글로벌)

```java
// presentation/filter/RateLimitFilter.java
@Component
@RequiredArgsConstructor
@Slf4j
public class RateLimitFilter extends OncePerRequestFilter {

    private final RateLimiter limiter;
    private final ObjectMapper objectMapper;

    // 정책 — endpoint 패턴 + 차원
    private static final List<RateLimitRule> RULES = List.of(
        new RateLimitRule("/api/v1/auth/login",                     Dimension.IP,   RateLimitPolicy.of(5,  Duration.ofMinutes(1))),
        new RateLimitRule("/api/v1/auth/signup",                    Dimension.IP,   RateLimitPolicy.of(3,  Duration.ofHours(1))),
        new RateLimitRule("/api/v1/auth/password-reset/request",    Dimension.IP,   RateLimitPolicy.of(3,  Duration.ofHours(1))),
        new RateLimitRule("/api/v1/auth/verify/email/request",      Dimension.USER, RateLimitPolicy.of(3,  Duration.ofHours(1))),
        new RateLimitRule("/api/v1/payments/confirm",               Dimension.USER, RateLimitPolicy.of(10, Duration.ofMinutes(1)))
    );

    private static final PathMatcher PATH = new AntPathMatcher();

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        var matched = RULES.stream()
            .filter(r -> PATH.match(r.pattern(), req.getServletPath()))
            .findFirst();
        if (matched.isPresent()) {
            var rule = matched.get();
            var key = buildKey(req, rule.dimension(), rule.pattern());
            var result = limiter.tryConsume(key, rule.policy());
            if (!result.allowed()) {
                writeRateLimitResponse(res, result.retryAfter());
                return;
            }
        }
        chain.doFilter(req, res);
    }

    private String buildKey(HttpServletRequest req, Dimension dim, String pattern) {
        var prefix = "rl:" + pattern + ":";
        return switch (dim) {
            case IP -> prefix + "ip:" + ClientIpUtil.resolveClientIp(req);
            case USER -> {
                var auth = SecurityContextHolder.getContext().getAuthentication();
                var userId = (auth != null && auth.isAuthenticated())
                    ? auth.getName() : "anonymous";
                yield prefix + "u:" + userId;
            }
            case IP_PLUS_USER -> {
                var auth = SecurityContextHolder.getContext().getAuthentication();
                var userId = (auth != null && auth.isAuthenticated())
                    ? auth.getName() : "anonymous";
                yield prefix + "ip:" + ClientIpUtil.resolveClientIp(req) + ":u:" + userId;
            }
        };
    }

    private void writeRateLimitResponse(HttpServletResponse res, Duration retryAfter) throws IOException {
        var body = CommonResponse.<Void>fail(ResponseCode.RATE_LIMIT_EXCEEDED,
            "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.");
        res.setStatus(HttpServletResponse.SC_TOO_MANY_REQUESTS);
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setCharacterEncoding(StandardCharsets.UTF_8.name());
        res.setHeader("Retry-After", String.valueOf(Math.max(1, retryAfter.toSeconds())));
        objectMapper.writeValue(res.getOutputStream(), body);
    }

    public record RateLimitRule(String pattern, Dimension dimension, RateLimitPolicy policy) {}
    public enum Dimension { IP, USER, IP_PLUS_USER }
}
```

`SecurityConfig` 에 등록:
```java
.addFilterBefore(rateLimitFilter, JwtAuthenticationFilter.class)
```

→ JWT 검증 전에 rate limit (DB 조회 비용 절약).

### 4.2 Annotation — `@RateLimit` (메서드 단)

```java
// presentation/annotation/RateLimit.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int capacity();
    int periodSeconds();
    Dimension dimension() default Dimension.USER;
    String keySuffix() default "";        // SpEL 식 ("#email") 도 가능
}
```

```java
// presentation/aspect/RateLimitAspect.java
@Aspect
@Component
@RequiredArgsConstructor
public class RateLimitAspect {

    private final RateLimiter limiter;
    private final SpelExpressionParser parser = new SpelExpressionParser();

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        var key = buildKey(pjp, rateLimit);
        var policy = RateLimitPolicy.of(rateLimit.capacity(),
                                        Duration.ofSeconds(rateLimit.periodSeconds()));
        var result = limiter.tryConsume(key, policy);
        if (!result.allowed()) {
            throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                "요청이 너무 많습니다.");
        }
        return pjp.proceed();
    }
    // buildKey — pjp 의 args + SpEL 평가
}
```

사용:
```java
@PostMapping("/login")
@RateLimit(capacity = 5, periodSeconds = 60, dimension = Dimension.IP_PLUS_USER, keySuffix = "#req.email")
public ResponseEntity<...> login(@RequestBody LoginRequest req) { ... }
```

### 4.3 Gateway / Nginx 단

운영에선 **API Gateway 가 1차 방어** 권장:
- Nginx `limit_req_zone` (간단)
- AWS API Gateway throttling
- Spring Cloud Gateway `RequestRateLimiter` filter
- Kong / Envoy

→ application 단 (위) 은 **세분화 정책** + **business-logic 의존** (e.g. user role 별 limit) 용.

---

## 5. SecurityConfig 등록

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        // ...
        .addFilterBefore(rateLimitFilter, JwtAuthenticationFilter.class)
        // ...
        .build();
}
```

순서: `ApiLogFilter` → `RateLimitFilter` → `JwtAuthenticationFilter`. Rate limit 가 JWT 검증보다 먼저 (DB lookup 비용 절약).

---

## 6. 응답 형식

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 45
Content-Type: application/json

{
  "code": "RATE_001",
  "status": "RATE_LIMIT_EXCEEDED",
  "message": "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요."
}
```

추가 헤더 (선택):
- `X-RateLimit-Limit: 5`
- `X-RateLimit-Remaining: 0`
- `X-RateLimit-Reset: 1715688000` (unix epoch)

---

## 7. user 역할 별 가변 정책

```java
public RateLimitPolicy policyFor(User user, String endpoint) {
    if (user.role() == Role.ADMIN) return RateLimitPolicy.of(1000, Duration.ofMinutes(1));
    if (user.role() == Role.SELLER) return RateLimitPolicy.of(100, Duration.ofMinutes(1));
    return RateLimitPolicy.of(60, Duration.ofMinutes(1));   // USER 기본
}
```

VIP / 결제 회원 등 변형 가능.

---

## 8. 함정 모음

### 함정 1 — IP 만 의존
NAT 뒤 여러 사용자 = 한 IP. 정상 트래픽 차단. **IP + user/email 조합** 또는 `X-Forwarded-For` 적절 처리.

### 함정 2 — `X-Forwarded-For` 신뢰
header 위조 가능. **trusted proxy** 만 신뢰. Nginx `real_ip_recursive` 설정.

### 함정 3 — Filter 안에서 무거운 DB 조회
모든 요청에 부하. **Bucket 상태는 Redis** + 빠른 조회.

### 함정 4 — Redis 다운 시 fail-open vs fail-close
Redis 죽으면 모든 요청 통과 (open) 또는 모두 차단 (close)? 보통 **fail-open** (정상 트래픽 우선) + 알람.

### 함정 5 — 분산 환경에서 in-memory rate limiter
JVM 마다 별도 카운터 = 실효성 X. **Redis 기반** 강제.

### 함정 6 — Token bucket 의 burst 허용
"100 requests/min" 인데 1초에 100 다 보냄. capacity 작게 + refill 짧게 (10/6s 등) 조절.

### 함정 7 — login 실패 후 success 도 카운트
공격자가 brute force 후 정상 user 도 차단. **실패만 카운트** 또는 success / fail 별도 bucket.

### 함정 8 — Filter chain 너무 늦음
JWT 검증 후 rate limit = DB 부하. **JWT 전에**.

### 함정 9 — `Retry-After` 누락
클라가 언제 재시도해야 할지 모름. 표준 헤더 채움.

### 함정 10 — 다중 인스턴스 + 분당 limit
인스턴스 마다 동기화 늦으면 실제 limit 초과. Bucket4j 의 distributed mode (Redis backed) 사용.

---

## 9. 운영 체크리스트

- [ ] Bucket4j 8.x + Redis backed
- [ ] Filter 가 JWT 보다 먼저
- [ ] `Retry-After` 헤더 포함
- [ ] Redis 다운 시 fail-open + 알람
- [ ] critical endpoint (login / payment) 별 정책 표 PR review
- [ ] X-Forwarded-For 신뢰 정책 (trusted proxy 만)
- [ ] role 별 limit 차등 (옵션)
- [ ] 429 비율 모니터링 — 너무 자주 발생 = 정책 검토

---

## 10. 관련

- [[login-jwt]] — login brute force 방어
- [[email-verification]] / [[password-reset]] — request rate
- [[distributed-lock]] — 비슷한 Redis 기반 패턴
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
