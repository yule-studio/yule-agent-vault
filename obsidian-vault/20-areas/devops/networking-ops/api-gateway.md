---
title: "API Gateway — Kong / AWS API GW / Apigee"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:46:00+09:00
tags: [devops, networking-ops, api-gateway]
---

# API Gateway — Kong / AWS API GW / Apigee

**[[networking-ops|↑ networking-ops]]**

---

## 1. 왜 필요

### 왜 필요
- MSA 환경 — backend 가 N 개. 클라이언트가 각 service 직접 호출 ❌.
- 횡단 관심사 (auth / rate-limit / log / cache) 를 한 곳에.
- 외부 공개 시 정책 일괄.

### 안 하면 문제
- 각 service 마다 auth / rate-limit 재구현.
- API 정책 분산 → 일관성 없음.
- backend 변경 시 client 모두 영향.

### 대안
- 단일 backend (monolith) — 단순.
- 클라이언트가 직접 service 호출 — coupling 강함.
- ALB + nginx — 일부 기능만 (auth / rate / transform 약함).

### 트레이드오프
- 추가 hop → latency.
- SPOF.
- 비용.

---

## 2. 핵심 기능

| 기능 | 예 |
| --- | --- |
| **routing** | path / version / header 기반 |
| **auth** | API key / JWT / OAuth2 / mTLS |
| **rate-limit** | per consumer / per IP |
| **transformation** | request/response 변환 |
| **caching** | response cache |
| **monitoring** | metric / log / trace |
| **versioning** | /v1, /v2 공존 |
| **deprecation** | sunset header |
| **canary** | traffic 분할 |

---

## 3. 도구 비교

| | type | 강점 | 약점 |
| --- | --- | --- | --- |
| **AWS API Gateway** | SaaS | Lambda 통합, Cognito | AWS lock-in, 비용 |
| **Kong** | OSS / SaaS | plugin 풍부, on-prem OK | OSS 운영 부담 |
| **Apigee (Google)** | SaaS | enterprise, analytics | 비싸, 무거움 |
| **Azure API Management** | SaaS | Azure 통합 | Azure lock-in |
| **Tyk** | OSS | dashboard | community 작음 |
| **Krakend** | OSS | 경량, response aggregator | 기능 적음 |
| **Envoy** | OSS proxy | mesh 와 통합 | API GW UI 약함 |
| **Spring Cloud Gateway** | Java | Spring 통합 | Java 만 |

---

## 4. AWS API Gateway 예

```
REST API:
  /users/{id}
    GET → Lambda getUser
    PUT → Lambda updateUser

  Authorizer: Cognito User Pool
  Stage: prod, dev
  Throttling: 1000 rps, burst 2000
  Cache: 5min, key=method+path
  CORS: Origin: https://app.example.com
```

```yaml
# CDK
const api = new RestApi(this, 'API')
const users = api.root.addResource('users')
users.addMethod('GET', new LambdaIntegration(getUserFn), {
    authorizer: new CognitoUserPoolsAuthorizer(this, 'auth', { cognitoUserPools: [pool] })
})
```

---

## 5. Kong 예

```yaml
# declarative config (kong.yaml)
_format_version: "3.0"
services:
  - name: user-service
    url: http://user.internal:8080
    routes:
      - name: users
        paths: [/api/v1/users]
        strip_path: false
    plugins:
      - name: rate-limiting
        config: { minute: 100, policy: redis }
      - name: jwt
        config: { secret_is_base64: true }
      - name: prometheus

consumers:
  - username: app-frontend
    jwt_secrets:
      - key: my-issuer
        secret: change-me
```

```bash
kong start -c kong.conf
deck sync --kong-addr http://localhost:8001
```

---

## 6. Spring Cloud Gateway

```kotlin
@Bean
fun routes(builder: RouteLocatorBuilder): RouteLocator = builder.routes()
    .route("user") { p ->
        p.path("/api/v1/users/**")
            .filters { f -> f
                .stripPrefix(2)
                .addRequestHeader("X-Source", "gateway")
                .circuitBreaker { c -> c.setName("user-cb").setFallbackUri("forward:/fallback") }
                .requestRateLimiter { r -> r.setRateLimiter(redisRateLimiter()) }
            }
            .uri("lb://user-service")
    }
    .build()
```

→ Spring 환경, 자체 Java 작성 시.

---

## 7. BFF (Backend For Frontend) 패턴

```
[mobile app] → [Mobile BFF] ─┐
                              ├─→ user, product, order, payment 서비스
[web app]    → [Web BFF]    ─┘
```

→ frontend 별 다른 API 형태 / aggregation. API Gateway 의 한 종류.

---

## 8. GraphQL Gateway

```
client → [GraphQL Gateway (Apollo)] → user, product, ... services
              ↓
            schema stitching / federation
```

→ over-fetching / under-fetching 해결. complexity ↑.

---

## 9. 권한 흐름 (auth)

```
client → API GW
            ├─ JWT 검증 (issuer, signature, expiry)
            ├─ scope 검사
            ├─ rate-limit (per consumer)
            ├─ log (audit)
            ▼
         backend (X-User-Id header 추가)
```

→ backend 는 JWT 재검증 X (gateway 신뢰), 또는 둘 다 (depth in defense).

---

## 10. 함정

1. **API GW 가 SPOF** — 무중단 deploy 필수, HA.
2. **너무 많은 logic gateway 에** — fat gateway → 변경 어려움.
3. **rate-limit per IP only** — 같은 IP 의 여러 사용자 영향.
4. **cache 잘못** — 사용자별 다른 응답이 같은 cache.
5. **auth 만 gateway → backend 보안 무신경** — internal 침투 시 무방비.
6. **version 정책 없음** — /v1 /v2 무한 누적.
7. **observability 부재** — 어디서 느린지 모름.

---

## 11. 관련

- [[networking-ops|↑ networking-ops]]
- [[service-mesh]]
- [[load-balancer-types]]
- [[../../60-recipes/spring/rate-limiting/rate-limiting|↗ rate-limit recipe]]
