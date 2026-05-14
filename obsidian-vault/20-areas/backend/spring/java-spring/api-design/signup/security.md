---
title: "signup §5 — 보안 / 인증·인가 / 암호화"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:42:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - security
---

# signup §5 — 보안 / 인증·인가 / 암호화

**[[signup|↑ signup hub]]**  ·  ← [[database]]  ·  → [[implementation]]

---

## 1. 인증 — 누가 이 API 를 호출하나

| 대상 | 호출 가능? | 이유 |
| --- | --- | --- |
| **anonymous (비로그인)** | ✅ | 가입 자체는 비인증 |
| 로그인된 user | ⚠️ | 이미 가입된 상태에서 가입 시도 = 이상 |
| 관리자 | △ | 관리자가 user 추가는 별도 endpoint (`/admin/users`) |

**결정**: `/api/v1/auth/signup` 은 **`permitAll`** + 단, 이미 로그인된 user 가 호출하면 새 user 가 생성됨 (현재 user 와 무관). 정책 옵션:
- (A) 받음 — 단순
- (B) 거절 — `이미 로그인 중`

본 레시피: **A** (단순). admin user 가 다른 사람 가입을 본인 PC 에서 할 수도 있음.

```java
// SecurityConfig.filterChain 의 authorizeHttpRequests 에
.requestMatchers("/api/v1/auth/signup", "/api/v1/auth/signup/**").permitAll()
```

## 2. 인가 — 권한 매트릭스

가입 endpoint 는 **권한 체크 불필요** (비인증).

후속 흐름 (가입 후 endpoint) 의 권한:

| Endpoint | 권한 |
| --- | --- |
| `POST /auth/signup` | anonymous |
| `POST /auth/verify/email/confirm` | anonymous (토큰만 검증) |
| `GET /me` | authenticated |
| `POST /auth/login` | anonymous |

## 3. 민감정보 처리 — "절대 X" 목록

| 항목 | 정책 |
| --- | --- |
| **평문 password** | DB / 로그 / 응답 어디에도 X. 입력 즉시 hash. |
| **password hash** | 응답에 노출 X. log 에도 X (Hibernate SQL `INFO` 이상). |
| **이메일** | 응답에는 OK. log 에는 마스킹 권장 (`a***e@x.com`). |
| **이름** | 응답에는 OK. log 에는 마스킹 권장. |
| **요청 body 전체** | `SignupRequest.toString()` 이 password 마스킹. (`***`) |

```java
public record SignupRequest(...) {
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s, termsAgreed=%s]"
            .formatted(email, name, termsAgreed);
    }
}
```

> **함정**: `data` / `record` 의 기본 `toString()` 은 모든 필드 포함. **password 가 들어가는 record 는 무조건 toString 오버라이드**.

## 4. 알고리즘 선정 — 패스워드 해시

| 알고리즘 | 권장 (2026) | 비고 |
| --- | --- | --- |
| **argon2id** | ✅ OWASP 1순위 | m=64MB t=3 p=4 |
| **bcrypt** | ✅ 대안 | cost ≥ 12. 72-byte 입력 제한 |
| scrypt | △ | argon2 보다 후순위 |
| PBKDF2-HMAC-SHA256 | △ | iterations ≥ 600,000 |
| MD5 / SHA-1 / SHA-256 / SHA-512 | ❌❌❌ | **패스워드 해시 X**. 초당 수십억 시도 가능 |

**결정**: **argon2id (password4j)** — m=64MB / t=3 / p=4 / salt 16 / out 32.

```java
@Component
public class Argon2PasswordEncoder implements PasswordEncoder {

    private static final int MEM_KB = 65536;   // 64 MB
    private static final int ITER   = 3;
    private static final int PAR    = 4;
    private static final int OUT_LEN = 32;
    private static final int SALT_LEN = 16;

    private final Argon2Function hasher =
        Argon2Function.getInstance(MEM_KB, ITER, PAR, OUT_LEN, Argon2.ID);

    @Override
    public String encode(String plain) {
        if (plain == null || plain.length() < 8 || plain.length() > 128)
            throw new IllegalArgumentException("password length out of range");
        return Password.hash(plain).addRandomSalt(SALT_LEN).with(hasher).getResult();
    }

    @Override
    public boolean matches(String plain, String hash) {
        return Password.check(plain, hash)
                       .with(Argon2Function.getInstanceFromHash(hash));
    }

    public boolean needsRehash(String hash) {
        var current = Argon2Function.getInstanceFromHash(hash);
        return current.getMemory() != MEM_KB
            || current.getIterations() != ITER
            || current.getParallelism() != PAR;
    }
}
```

> **bcrypt 의 72-byte 한계**: 긴 passphrase 가 silently truncated. 본 레시피는 argon2 라 무관.

## 5. 패스워드 정책 — NIST 800-63B (2017+)

| 항목 | 정책 |
| --- | --- |
| 최소 길이 | **8자** (NIST 권장: 길수록 좋음) |
| 최대 길이 | **128 자** 허용 (자르지 말 것) |
| 복잡도 (대소문자 / 숫자 / 특수문자 강제) | **X** — 길이 우선 |
| 알려진 유출 패스워드 차단 | **권장** (haveibeenpwned k-anonymity) |
| 사용자명 / 이메일 포함 차단 | 권장 |

```java
@Component
public class PasswordPolicy {

    private final PwnedPasswordChecker pwned;     // optional

    public void validate(String plain) {
        if (plain == null || plain.length() < 8 || plain.length() > 128)
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password length must be 8-128");
        if (plain.contains(" "))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password cannot contain spaces");
        if (pwned != null && pwned.isPwned(plain))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password found in known breach");
    }
}
```

## 6. enumeration 방어 — 이메일 존재 여부 노출

**현재 정책**: 중복 이메일이면 409 + "이미 가입된 이메일" 응답 (직관적 UX 우선).

**대안 (보안 민감 도메인)**: **항상 200 + "이메일을 확인해 주세요" 동일 응답**. 실제 가입 / 안 가입은 backend 에서 분기. 클라는 차이를 모름.

| 정책 | UX | 보안 |
| --- | --- | --- |
| 직관적 (409) | ✅ | ⚠️ 가입자 enumeration |
| Enumeration 차단 (항상 200) | ⚠️ 친절도 ↓ | ✅ |

본 레시피: **직관적** (일반 SaaS). 금융 / 보건 도메인은 enumeration 차단.

## 7. Rate limiting

```
POST /auth/signup
  - IP 별 1시간 3회
  - 같은 이메일 1일 1회
```

[[../rate-limiting]] 의 Bucket4j + Redis 적용. SecurityConfig 의 filter chain 에 RateLimitFilter 등록.

## 8. OWASP Top 10 (2021) 매핑

| OWASP | 본 레시피 대응 |
| --- | --- |
| A01 Broken Access Control | endpoint 공개 정책 명확 (위 §2) |
| A02 Cryptographic Failures | argon2id + salt + iteration |
| A03 Injection | jakarta.validation + JPA parameterized query |
| A04 Insecure Design | 도메인 모델 + 상태머신 |
| A05 Security Misconfiguration | Spring Security STATELESS + CORS + CSP |
| A07 Identification & Auth | enumeration 정책 + rate limit |
| A08 Software & Data Integrity | argon2 PHC 형식 검증 |
| A09 Security Logging Failures | 가입 / 실패 audit log |

## 9. 운영 관련

- **HTTPS 강제** — production endpoint 는 HTTPS 만 (HSTS 헤더 + ELB / WAF 강제)
- **TLS 1.2+ 만** — 옛 TLS 차단
- **HSTS** — `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- **`SecureHeaders`** — `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, CSP
- **WAF / CAPTCHA** — 봇 가입 대량 시도 방어

## 10. 관련

- [[signup|↑ signup hub]]
- [[database]] — 이전 (§4)
- [[implementation]] — 다음 (§6)
- [[../../common/security-config]] — 표준 SecurityConfig 패턴
- [[../two-factor-auth]] — 가입 후 2FA 추가
- [[../oauth2-social-login]] — 소셜 가입
