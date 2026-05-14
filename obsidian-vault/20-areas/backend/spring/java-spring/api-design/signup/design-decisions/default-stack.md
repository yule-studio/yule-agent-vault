---
title: "본 vault 의 기본 stack — 종합 결과"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - default-stack
---

# 본 vault 의 기본 stack — 종합 결과

**[[design-decisions|↑ design-decisions hub]]**

> 13 개 결정의 종합. 새 프로젝트 시작 시 `application.yml` / config 로 그대로 사용.

---

## 1. 종합 — 한 페이지

```yaml
auth:
  # ─── 패스워드 ──────────────────────────────────────
  password-hash:
    algorithm: argon2id
    params: { m: 65536, t: 3, p: 4 }     # 64MB / 3 iter / 4 parallel
    needs-rehash: true                    # 로그인 시 점진 강화
  password-policy:
    min-length: 8
    max-length: 128
    pwned-check: true                     # haveibeenpwned

  # ─── JWT ─────────────────────────────────────────
  jwt:
    algorithm: HS256                      # 모놀리식. MSA = RS256
    access-ttl: 15m
    refresh-ttl: 14d
    refresh-rotation: true
    refresh-storage: rdb                  # PostgreSQL refresh_tokens
    reuse-detection: enabled

  # ─── 가입 인증 ────────────────────────────────────
  signup-verification:
    channels: [email, phone]              # 한국 SaaS 기본
    email-provider: aws-ses
    sms-provider: ncp-sens                # AlimTalk fallback
    identity-verification: optional       # 금융 / 의료만

  # ─── 소셜 로그인 ─────────────────────────────────
  social-login:
    providers: [kakao, naver, google, apple]
    email-link-policy: verified-only      # email_verified_at 있을 때만 link

  # ─── 2FA ──────────────────────────────────────────
  two-factor-auth:
    enforcement: optional + step-up
    admin: required
    method: totp
    recovery-codes: 10

  # ─── 약관 / 동의 ─────────────────────────────────
  terms:
    versioning: enabled
    re-consent-on-change: required
    pre-notice-days: 30                   # 한국 정보통신망법

  # ─── 정책 ─────────────────────────────────────────
  enumeration: explicit                   # B2C 일반. 금융 = obscured
  idempotency: optional                   # 결제 / 주문 = required

  # ─── Rate Limit ───────────────────────────────────
  rate-limit:
    signup: 3/hour per IP
    login: 5/min per IP+email
    password-reset: 3/hour per IP
    sms-request: 5/hour per phone + 60s cooldown
    email-verification-resend: 3/hour per user

  # ─── 보안 ─────────────────────────────────────────
  security:
    csrf: enabled (cookie + header)
    cors: explicit origin list
    https-only: true
    hsts: max-age=31536000
    timing-attack-defense: dummy-hash on user-not-found
```

---

## 2. 환경 변수 (`.env` 또는 KMS)

```bash
# JWT
AUTH_JWT_SECRET=<base64-256bit>             # 256-bit random
AUTH_JWT_ISSUER=yule-studio

# DB
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/yule
SPRING_DATASOURCE_USERNAME=yule
SPRING_DATASOURCE_PASSWORD=<KMS>

# Redis
SPRING_DATA_REDIS_HOST=localhost
SPRING_DATA_REDIS_PORT=6379

# AWS SES
AWS_REGION=ap-northeast-2
AWS_ACCESS_KEY_ID=<KMS>
AWS_SECRET_ACCESS_KEY=<KMS>
AWS_SES_FROM=noreply@example.com

# NCP SENS
NCP_SENS_SERVICE_ID=<KMS>
NCP_SENS_ACCESS_KEY=<KMS>
NCP_SENS_SECRET=<KMS>
NCP_SENS_FROM=01012345678

# Apple Sign In (옵션)
APPLE_CLIENT_ID=com.example.shop
APPLE_TEAM_ID=ABC123
APPLE_KEY_ID=DEF456
APPLE_PRIVATE_KEY=<base64>                  # KMS

# Kakao / Naver / Google
KAKAO_CLIENT_ID=<KMS>
NAVER_CLIENT_ID=<KMS>
GOOGLE_CLIENT_ID=<KMS>
```

---

## 3. application.yml

```yaml
spring:
  application:
    name: yule-studio
  profiles:
    active: prod
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    hikari:
      maximum-pool-size: 20
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        jdbc:
          time_zone: UTC
  data:
    redis:
      host: ${SPRING_DATA_REDIS_HOST}
      port: ${SPRING_DATA_REDIS_PORT}
  flyway:
    enabled: true
    locations:
      - classpath:db/migration
      - classpath:db/migration_${spring.profiles.active}

server:
  port: 8080
  forward-headers-strategy: framework
  compression:
    enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true

logging:
  level:
    com.example: INFO
    org.springframework.security: INFO
  pattern:
    correlation: "[%X{traceId:-},%X{spanId:-}]"

# auth 도메인 설정 (위 §1 참고)
auth:
  jwt:
    secret: ${AUTH_JWT_SECRET}
    issuer: ${AUTH_JWT_ISSUER}
    access-ttl: PT15M
    refresh-ttl: P14D
  password:
    argon2:
      memory: 65536
      iterations: 3
      parallelism: 4
```

---

## 4. 시작 체크리스트

### 4.1 인프라

- [ ] PostgreSQL 16 (서울 region, encryption at rest)
- [ ] Redis 7 (옵션 — rate limit / 캐시)
- [ ] AWS SES (서울 region, sandbox 해제)
- [ ] NCP SENS (계정 + 발신번호 등록)
- [ ] AlimTalk 템플릿 (선택 — 검수 1주)
- [ ] DNS — SPF / DKIM / DMARC

### 4.2 보안

- [ ] JWT secret 256-bit random (`openssl rand -base64 32`)
- [ ] KMS 또는 secret manager 사용
- [ ] HTTPS / TLS 1.2+
- [ ] CORS 화이트리스트
- [ ] WAF (AWS WAF / Cloudflare) — DDoS 방어

### 4.3 모니터링

- [ ] CloudWatch / Datadog / Grafana
- [ ] Alert — login failure rate, signup abuse, SES bounce rate
- [ ] Sentry (or 비슷한) — application error

### 4.4 운영

- [ ] Flyway 마이그레이션 자동 (dev/staging) + 수동 (prod)
- [ ] Backup — RDS automated snapshot, point-in-time recovery
- [ ] Disaster Recovery — 다중 AZ
- [ ] Log retention — 30일 (CloudWatch) + 1년 (S3)

---

## 5. 다른 컨텍스트별 변경

### 5.1 B2B SaaS

```yaml
auth:
  signup-verification:
    channels: [email]                     # phone X
  social-login:
    providers: [google, microsoft]
  jwt:
    algorithm: RS256                       # MSA 가능성
  rate-limit:
    signup: relaxed
```

### 5.2 금융 / 의료

```yaml
auth:
  password-hash:
    params: { m: 131072, t: 4, p: 4 }     # 강화
  signup-verification:
    channels: [email, phone]
    identity-verification: required        # PASS / NICE
  two-factor-auth:
    enforcement: required-at-signup
  enumeration: obscured
  idempotency: required (all)
```

### 5.3 글로벌

```yaml
auth:
  signup-verification:
    email-provider: aws-ses-multi-region
    sms-provider: twilio
  social-login:
    providers: [google, facebook, apple]
  terms:
    gdpr: enabled
```

---

## 6. 검증된 운영 메트릭 — 본 vault stack

| 메트릭 | 목표 (p99) | 알람 |
| --- | --- | --- |
| Signup 응답 시간 | < 1s | > 3s |
| Login 응답 시간 | < 500ms | > 2s |
| Token refresh | < 100ms | > 500ms |
| Email 발송 (outbox → SES) | < 5min | > 30min |
| SMS 발송 | < 30s | > 2min |
| Login failure rate | < 5% | > 20% |
| Signup abuse rate | < 0.1% | > 1% |

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[dependencies]] — 코드 의존성
- [[../signup-impl]] · [[../login-impl]] · [[../password-reset-impl]] — 구현
- [[../security]] — 보안 깊이
- [[../operations]] — 운영
