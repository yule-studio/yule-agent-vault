---
title: "보안 / 비밀번호 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - pitfalls
  - security
---

# 보안 / 비밀번호 함정

**[[pitfalls|↑ pitfalls hub]]**

> 비밀번호 / 토큰 / 평문 데이터 처리의 흔한 실수.

---

## 함정 1 — SHA-256 / MD5 로 password 해시

### 무엇

```java
String hashed = sha256(password);    // ❌
```

### 왜 위험

- GPU 로 초당 수십억 회 시도 가능.
- Rainbow table — 1억 비밀번호의 hash 가 미리 계산된 DB 공유.
- 사실상 평문 저장과 동급.

### 해결

- argon2id (1순위) / bcrypt cost ≥ 12 (대안).
- 자세히: [[../design-decisions/password-hash]].

---

## 함정 2 — 평문 password 가 로그에 노출

### 무엇

```java
public record SignupRequest(String email, String password, String name) { }

@Slf4j
class Controller {
    @PostMapping("/signup")
    public Response signup(@RequestBody SignupRequest req) {
        log.info("signup request: {}", req);    // ❌ password 평문 출력
    }
}
```

**문제**
- record 의 기본 toString = 모든 필드 출력.
- 로그 유출 = 모든 평문 password 노출.

### 왜 위험

- CloudWatch / Sentry / log aggregator 유출.
- 한국 정보보호법 위반 + 사용자 신뢰 손상.

### 해결

```java
public record SignupRequest(String email, String password, String name) {
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s]"
            .formatted(email, name);
    }
}
```

자세히: [[../security/sensitive-data-handling]].

---

## 함정 3 — Hibernate SQL 로그가 password_hash 노출

### 무엇

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG          # ❌ 운영
```

**문제**
- SQL parameter 가 그대로 출력 — INSERT 시 password_hash 가 로그에.
- DEBUG level + log aggregator = hash 유출.

### 왜 위험

- hash 도 brute force 가능 (약한 password 면).
- 부분적 보안 약화.

### 해결

- prod = WARN+.
- dev DEBUG OK (dev DB 는 가짜 데이터).

---

## 함정 4 — bcrypt 의 72-byte 컷

### 무엇

```java
String hashed = BCrypt.hashpw(longPassphrase, BCrypt.gensalt(12));
```

**문제**
- bcrypt 입력 max 72 byte.
- 73+ byte 는 silent truncate.
- 사용자가 길게 입력해도 앞 72 만 검증.

### 왜 위험

- "correct horse battery staple ..." passphrase 사용자 보호 약화.
- silent — 사용자 / 운영자 모름.

### 해결

- argon2id (제한 없음).
- 또는 bcrypt + SHA-256 pre-hash (Dropbox 방식):
  ```java
  String preHash = base64(sha256(password));
  String bcryptHash = BCrypt.hashpw(preHash, salt);
  ```

---

## 함정 5 — Bean Validation 만으로 약한 password 차단

### 무엇

```java
public record SignupRequest(@Size(min = 8) String password, ...) { }
```

**문제**
- `"12345678"` 통과 — 8자 충족.
- "password" 통과 — 알려진 유출 비번.

### 왜 위험

- 즉시 brute force.
- credential stuffing 즉시 탈취.

### 해결

```java
@Component
public class PasswordPolicy {
    public void validate(String plain) {
        if (plain.length() < 8 || plain.length() > 128) throw ...;
        if (pwnedChecker.isPwned(plain)) throw ...;
    }
}
```

자세히: [[../security/password-policy]].

---

## 함정 6 — 응답에 password / password_hash 노출

### 무엇

```java
@GetMapping("/me")
public UserJpaEntity me(...) { return user; }    // ❌ password_hash 포함
```

### 왜 위험

- JSON 응답에 hash 노출.
- 클라 / 캐시 / 백업 어디서든 유출.

### 해결

```java
@GetMapping("/me")
public UserResponse me(...) {
    return new UserResponse(user.id(), user.email(), user.name(), user.status());
    // password_hash 명시 제외
}
```

+ Entity 에 `@JsonIgnore`:
```java
@JsonIgnore
@Column(name = "password_hash") private String passwordHash;
```

---

## 함정 7 — JWT secret hardcoded

### 무엇

```java
private static final String SECRET = "my-secret-key";    // ❌
```

### 왜 위험

- git history 영구 노출.
- 누구나 JWT 위조 가능 → 모든 계정 탈취.

### 해결

- KMS / AWS Secrets Manager.
- application.yml + env:
  ```yaml
  auth:
    jwt:
      secret: ${AUTH_JWT_SECRET}
  ```

자세히: [[../operations/configuration-secrets]].

---

## 함정 8 — JWT secret 약함

### 무엇

```bash
AUTH_JWT_SECRET="secret"          # ❌
```

### 왜 위험

- 1초 안에 brute force.
- HS256 의 256-bit 강제 미충족.

### 해결

```bash
AUTH_JWT_SECRET="$(openssl rand -base64 32)"
```

256-bit (32 byte) random.

---

## 함정 9 — `alg: "none"` 라이브러리 취약점

### 무엇

```java
Jwts.parser().parseClaimsJws(token)         // alg=none 허용?
```

2015 사고 다수 — JWT 의 `alg` claim 을 `"none"` 으로 위조 → 서명 검증 우회.

### 왜 위험

- 옛 라이브러리는 `alg=none` 통과.

### 해결

- 최신 jjwt 0.12+ (기본 거부).
- 명시 옵션:
  ```java
  Jwts.parser()
      .verifyWith(secretKey)
      .build()
      .parseSignedClaims(token);    // signed 강제
  ```

---

## 함정 10 — Refresh token raw 저장

### 무엇

```sql
INSERT INTO refresh_tokens (id, token, ...) VALUES (?, ?, ...);
                                  ^^^^^ raw
```

### 왜 위험

- DB 유출 = 모든 RT 즉시 사용.
- 14일 동안 모든 사용자 탈취.

### 해결

```sql
INSERT INTO refresh_tokens (id, token_hash, ...) VALUES (?, sha256(?), ...);
```

자세히: [[../database/refresh-tokens-table#2.3]].

---

## 함정 11 — Reuse detection 없음

### 무엇

- ROTATED 상태 RT 가 다시 들어옴 = 도난 신호.
- 무시하면 → 공격자 / user 양쪽 다 사용 가능.

### 왜 위험

- 도난 감지 X → 영향 시간 ↑.
- chain 분석 X.

### 해결

```java
if (rt.status() == ROTATED) {
    refreshTokens.revokeChain(rt.id());        // chain 전체 revoke
    alarmService.notifySuspiciousActivity(rt.userId());
    throw new SuspiciousReuseException();
}
```

자세히: [[../database/refresh-tokens-table#3 Rotation Chain]].

---

## 함정 12 — Account lock 없음

### 무엇

- Login 실패 무제한 → 사실상 brute force 가능.

### 왜 위험

- argon2id (200ms) 로도 분당 300 시도 → 1년 brute force 가능.

### 해결

```java
if (failureCount(userId, last15min) >= 5) {
    throw new AccountLockedException();
}
```

5회 / 15분 lock.

---

## 함정 13 — Timing attack 무방비

### 무엇

```java
public boolean login(String email, String password) {
    var user = users.findByEmail(email).orElse(null);
    if (user == null) return false;                        // 5ms (빠름)
    return encoder.matches(password, user.passwordHash()); // 200ms (느림)
}
```

**문제**
- 응답 시간 차이 = user 존재 enumeration.

### 해결

```java
if (user == null) {
    encoder.matches(password, DUMMY_HASH);     // 같은 시간 소비
    return false;
}
```

자세히: [[../security/attack-defense#5 Timing]].

---

## 함정 14 — String.equals 토큰 비교

### 무엇

```java
if (token.equals(expected)) ...               // ❌ early return
```

**문제**
- 첫 글자 다르면 즉시 false → byte 단위 timing attack.

### 해결

```java
MessageDigest.isEqual(tokenBytes, expectedBytes)   // constant-time
```

---

## 함정 15 — Recovery code 없이 2FA 강제

### 무엇

- 2FA 활성 후 recovery code 발급 X.
- 단말 분실 = 영구 잠금.

### 해결

- 활성 시 10 개 recovery code.
- DB 에 hash 저장.
- 사용 시 status=USED 전이.

자세히: [[../design-decisions/two-factor-auth-policy#5]].

---

## 함정 16 — TOTP secret 평문 저장

### 무엇

```sql
totp_secret VARCHAR(32) NOT NULL    -- 평문
```

### 왜 위험

- DB 유출 = 모든 2FA 우회.

### 해결

- 컬럼 암호화 (pgcrypto 또는 application AES).
- 또는 별도 KMS-managed vault.

---

## 함정 17 — Password 변경 후 refresh revoke 누락

### 무엇

```java
public void changePassword(...) {
    user.changePassword(...);
    users.save(user);
    // refresh tokens 그대로!
}
```

**문제**
- 공격자가 옛 refresh 로 계속 access token 갱신.
- 비밀번호 변경 의미 X.

### 해결

```java
public void changePassword(...) {
    user.changePassword(...);
    users.save(user);
    refreshTokens.revokeAllForUser(user.id(), "PASSWORD_CHANGED");
}
```

---

## 함정 18 — IP 추출 잘못 (X-Forwarded-For)

### 무엇

```java
String ip = request.getRemoteAddr();    // CDN / ALB IP
```

**문제**
- ALB / CloudFront 뒤 → 모든 사용자가 같은 IP.
- audit / rate limit 망함.

### 해결

```yaml
server:
  forward-headers-strategy: framework
```

또는 명시:
```java
String ip = request.getHeader("X-Forwarded-For");
if (ip != null) ip = ip.split(",")[0].trim();
```

---

## 함정 19 — Sentry / Datadog 의 PII 자동 수집

### 무엇

- Sentry breadcrumb / context 자동 수집 — request body 포함.
- 평문 password 가 Sentry 에.

### 해결

```yaml
sentry:
  send-default-pii: false
```

+ scrubbing rules.

---

## 함정 20 — Heap dump 가 secret 포함

### 무엇

- OOM 시 JVM heap dump 자동 생성.
- 메모리 안의 평문 password / JWT 모두 dump.

### 해결

- prod heap dump 제한 (또는 즉시 폐기 정책).
- secret 은 byte[] (가능하면 즉시 zero-out).

---

## 관련

- [[pitfalls|↑ pitfalls hub]]
- [[../security/sensitive-data-handling]] · [[../security/attack-defense]]
- [[../design-decisions/password-hash]] · [[../design-decisions/token-model]]
