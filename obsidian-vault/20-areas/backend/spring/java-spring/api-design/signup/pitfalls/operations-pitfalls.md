---
title: "UX / 정책 / 운영 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - pitfalls
  - ux
  - policy
  - operations
---

# UX / 정책 / 운영 함정

**[[pitfalls|↑ pitfalls hub]]**

> 코드 / 로직 외 정책 / UX / 운영 의 흔한 실수.

---

## 함정 1 — Enumeration 무방비

### 무엇

```java
// 잘못된 응답 분기
if (users.existsByEmail(email)) return 409 "이미 가입된 이메일";
// 가입 안 됐으면 200
```

### 왜 위험

- 공격자가 자동화 스크립트로 가입자 list enumeration.
- 가입자 list = phishing 표적.

### 해결

**일반 SaaS**: 직관적 (409) + Rate limit + CAPTCHA.

**금융 / 의료**: 응답 통일 ("메일을 확인하세요" 동일).

자세히: [[../design-decisions/enumeration-policy]].

---

## 함정 2 — 가입 직후 ACTIVE

### 무엇

```java
public User signup(...) {
    var user = User.register(...);
    user.activate();              // ❌ 즉시 ACTIVE
    return users.save(user);
}
```

### 왜 위험

- 이메일 인증 없이 ACTIVE = 가짜 메일 가입 가능.
- 봇 / spam 가입 폭주.

### 해결

```java
public User signup(...) {
    var user = User.register(...);     // status = PENDING_VERIFICATION
    users.save(user);
    // 이메일 인증 후 ACTIVE
    return user;
}
```

---

## 함정 3 — 가입 직후 자동 로그인 (JWT 발급)

### 무엇

```java
public SignupResponse signup(...) {
    var user = ...;
    var jwt = tokenProvider.generate(user.id());
    return new SignupResponse(user.id(), jwt);    // ❌
}
```

### 왜 위험

- 가입과 로그인 결합.
- 이메일 인증 흐름과 충돌 (PENDING user 도 로그인됨).
- 보안 약화 (인증 안 된 user 가 API 사용).

### 해결

- 가입 = user 생성만.
- 로그인 = 별도 API + 인증 후.

---

## 함정 4 — Idempotency-Key 무시 (결제 / 가입)

### 무엇

```java
@PostMapping("/payment")
public Response pay(@RequestBody PaymentRequest req) {
    // Idempotency-Key 없이 처리
}
```

### 왜 위험

- 네트워크 retry = 중복 결제.
- 한 사용자 = 2번 차감.

### 해결

- 결제 / 주문 = Required.
- 일반 mutating = Optional.

자세히: [[../design-decisions/idempotency-policy]].

---

## 함정 5 — 약관을 `users.marketing_agreed BOOLEAN` 으로

### 무엇

```sql
ALTER TABLE users ADD COLUMN marketing_agreed BOOLEAN;
```

### 왜 위험

- 약관 버전 변경 추적 X.
- 동의 history (동의 → 철회 → 재동의) X.
- 한국 정보통신망법 §50 위반 (필수+선택 분리 부재).

### 해결

- 별도 `terms` + `user_terms_consent_history` 테이블.
- 자세히: [[../database/terms-tables]] · [[../design-decisions/terms-consent-policy]].

---

## 함정 6 — Rate Limit 없음

### 무엇

- 가입 / 로그인 / sms 인증 endpoint 에 rate limit 없음.

### 왜 위험

- 봇이 무한 시도 → 인프라 부담 + SMS / SES 비용 폭증.
- argon2id 검증 비용 (200ms × 1000회 = 200s CPU).

### 해결

- Bucket4j + Redis.
- IP + user 별 limit.

자세히: [[../../rate-limiting]] · [[../security/attack-defense#10]].

---

## 함정 7 — JwtAuthenticationEntryPoint 의 응답 status 잘못

### 무엇

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res, AuthenticationException ex) {
        res.setStatus(HttpServletResponse.SC_BAD_REQUEST);    // ❌ 400
    }
}
```

### 왜 위험

- 인증 실패 = 401 인데 400 응답.
- 클라가 401 분기 X — 자동 logout 흐름 안 작동.

### 해결

```java
res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);    // 401
```

(실제 사례: job-answer-be 의 버그 — [[../../common/security-config#4.1]]).

---

## 함정 8 — Outbox 워커 다운 미감지

### 무엇

- email outbox 워커 다운 → 인증 메일 무한 큐.
- 메트릭 / 알람 없음 → CS 폭주 후 발견.

### 해결

```yaml
# Prometheus
- alert: EmailOutboxBacklog
  expr: email_outbox_pending > 1000
  for: 5m
```

자세히: [[../operations/observability]].

---

## 함정 9 — 모든 사용자에게 같은 from 주소

### 무엇

- 가입 / 마케팅 / 결제 영수증 모두 `noreply@example.com`.

### 왜 위험

- 한 메일 spam 폴더 직행 = 모든 메일 평판 영향.
- 마케팅 complaint 가 트랜잭션 메일에 영향.

### 해결

- 서브도메인 분리:
  - `auth@auth.example.com` (가입 / 인증)
  - `marketing@mail.example.com` (마케팅)
  - `receipt@billing.example.com` (영수증)

자세히: [[../design-decisions/email-provider#7]].

---

## 함정 10 — SPF / DKIM / DMARC 누락

### 무엇

- DNS 에 SPF / DKIM 만 설정.

### 왜 위험

- Gmail / Naver / Daum 이 spam 폴더 직행.
- 사용자가 인증 메일 못 받음.

### 해결

- 3개 모두 설정.
- DMARC = `p=quarantine` 시작 → 안정 후 `p=reject`.

자세히: [[../design-decisions/email-provider#4]].

---

## 함정 11 — User-Agent / IP 의 trust 너무 강함

### 무엇

```java
String ip = request.getHeader("X-Forwarded-For");   // 그대로 신뢰
```

### 왜 위험

- 공격자가 임의로 X-Forwarded-For 설정.
- 의심 활동 분석 / rate limit 우회.

### 해결

- ALB / Cloudflare 뒤에서만 — 신뢰할 수 있는 proxy.
- Spring `forward-headers-strategy: framework` 사용.
- 첫 IP 만 신뢰 (CDN edge 의 client IP).

---

## 함정 12 — Privacy Policy / 약관 변경 시 사전 공지 없음

### 무엇

- 약관 변경 즉시 효력.

### 왜 위험

- 한국 정보통신망법 §27의2 위반 (30일 사전 공지 의무).
- 변경 효력 무효.

### 해결

- effective_at = now() + 30일.
- 안내 메일 / 화면 모달.

자세히: [[../design-decisions/terms-consent-policy#3.4]].

---

## 함정 13 — 모든 응답이 같은 ResponseCode

### 무엇

```java
throw new BusinessException(ResponseCode.BAD_REQUEST);    // 모든 에러
```

### 왜 위험

- 클라가 분기 어려움.
- CS 응대 시 원인 파악 어려움.

### 해결

- ResponseCode 세분화:
  - `EMAIL_ALREADY_EXISTS`
  - `INVALID_PASSWORD_FORMAT`
  - `TERMS_NOT_AGREED`
  - `USER_LOCKED`
  - `USER_NOT_FOUND` 또는 `INVALID_CREDENTIALS` (enumeration 정책 따라)

자세히: [[../../common/response-envelope]].

---

## 함정 14 — Swagger UI 가 prod 노출

### 무엇

```yaml
springdoc:
  swagger-ui:
    enabled: true             # 모든 환경
```

### 왜 위험

- API 구조 / endpoint list 가 공격자에게 노출.
- 일부는 / actuator 같은 민감 endpoint 도.

### 해결

```yaml
springdoc:
  swagger-ui:
    enabled: ${SWAGGER_ENABLED:false}    # dev / staging 만
```

또는 IP 화이트리스트.

---

## 함정 15 — 모든 사용자 정보가 응답에

### 무엇

```java
@GetMapping("/admin/users")
public List<UserJpaEntity> all() { return userRepo.findAll(); }
```

### 왜 위험

- pagination 없음 → OOM.
- password_hash / phone / 모든 PII 노출.

### 해결

- DTO + Page.
- 명시 필드만.

---

## 함정 16 — 비밀번호 입력에 autocomplete="off"

### 무엇

```html
<input type="password" autocomplete="off" />
```

### 왜 위험

- 비밀번호 매니저 (1Password / Bitwarden) 자동완성 X.
- 사용자가 약한 비밀번호 선택.

### 해결

```html
<input type="password" autocomplete="new-password" />     <!-- 가입 -->
<input type="password" autocomplete="current-password" /> <!-- 로그인 -->
```

---

## 함정 17 — Health check 가 단순 200

### 무엇

```java
@GetMapping("/health")
public String health() { return "OK"; }
```

### 왜 위험

- 앱 시작 직후 = DB 연결 안 됐는데 200.
- ALB 가 traffic 보내고 5xx 폭증.

### 해결

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: db, redis, livenessState
```

- Spring Boot Actuator 의 readiness / liveness 분리.

---

## 함정 18 — Graceful shutdown 없음

### 무엇

```yaml
# 옛 default
server:
  shutdown: immediate
```

### 왜 위험

- 배포 시 in-flight 요청 cut.
- 사용자 5xx 응답.

### 해결

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

## 관련

- [[pitfalls|↑ pitfalls hub]]
- [[../design-decisions/enumeration-policy]] · [[../design-decisions/idempotency-policy]] · [[../design-decisions/email-provider]]
- [[../operations/observability]] · [[../operations/runbook]]
- [[../../common/security-config]] · [[../../common/response-envelope]]
