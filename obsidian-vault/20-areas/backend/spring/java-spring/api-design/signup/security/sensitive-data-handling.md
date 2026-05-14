---
title: "민감정보 처리 — 로그 / 응답 / DB / toString"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - pii
  - logging
---

# 민감정보 처리 — 로그 / 응답 / DB / toString

**[[security|↑ security hub]]**

> 평문 password, PII (email, phone, name) 가 어디까지 노출돼도 되나. 무심코 `log.info(request)` 한 줄이 **전체 사용자 정보 유출**.

---

## 1. 민감도 분류

| 분류 | 데이터 | 어디 노출 OK |
| --- | --- | --- |
| **최고 (절대 노출 X)** | 평문 password, JWT secret, KMS key | 메모리에서만, 즉시 폐기 |
| **높음 (백엔드만)** | password_hash, refresh_token raw, recovery_code | DB 만, 로그/응답 X |
| **중간 (마스킹 권장)** | email, phone (평문), name | 로그는 마스킹, 응답 OK |
| **낮음 (그대로)** | user_id (ULID), status, role, created_at | 모두 OK |

---

## 2. "절대 X" 목록 — 4구조

### 2.1 평문 password — 로그 / 응답 / DB

**왜 절대 X**
- DB / 로그 / 응답 어디든 평문 → 한 곳 유출 = 모든 계정 탈취.
- 한국 정보보호법 / GDPR 위반.

**안 하면 무슨 문제**
- 응답에 password 포함 → 클라 캐싱 / 화면 노출 / 백업 유출 = 사고.
- 로그에 password 포함 → CloudWatch / Sentry / log aggregator 유출 시.
- Hibernate SQL 로그 (`spring.jpa.show-sql=true` + DEBUG) 가 password 출력.

**대안과 왜 안 됨**
- 마스킹 (`***`) 만 → 일부 케이스 누락 위험.
- 암호화 후 저장 → key 분실 시 모든 password 복구 가능 = 안 됨. 일방향 hash 만.

**구현**
```java
// 1. 입력 즉시 hash
@Service
public class UserService {
    public User signup(SignupCommand cmd) {
        String rawPassword = cmd.password();
        String hashed = encoder.encode(rawPassword);
        var hash = new PasswordHash(hashed);
        // rawPassword 는 method scope 안에서만 — 함수 끝나면 GC
        var user = User.register(..., hash, ...);
        return user;
    }
}

// 2. SignupRequest 의 toString 마스킹
public record SignupRequest(String email, String password, String name) {
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s]".formatted(email, name);
    }
}
```

**왜 toString 오버라이드 필수**
- record 의 기본 toString 이 모든 필드 출력.
- `log.info("request: {}", request)` 가 자동으로 toString 호출.
- 마스킹 안 하면 → password 가 로그에 그대로.

---

### 2.2 password_hash — 응답 / 로그

**왜 안 됨**
- hash 도 brute force 가능 (argon2 라도 약한 password 면).
- 응답 / 로그 노출 시 부분적 보안 약화.

**구현**
- 응답 DTO 에 passwordHash 필드 없음 (명시적 작성).
- `User` 도메인의 `passwordHash()` 접근자는 internal 만.
- JPA Entity 의 `passwordHash` 컬럼 — `@JsonIgnore`.

```java
@Entity
public class UserJpaEntity {
    @JsonIgnore                          // 직렬화 시 제외
    @Column(name = "password_hash")
    private String passwordHash;
}
```

---

### 2.3 JWT secret / KMS key — 코드 / 환경변수

**왜 안 됨**
- 코드에 hardcoded → git 유출 = 즉시 사고.
- env 평문 → process list / heap dump 유출.

**구현**
- KMS / AWS Secrets Manager / HashiCorp Vault.
- env 는 부팅 시점에 fetch.
- application.yml 안에 직접 X.

```yaml
auth:
  jwt:
    secret: ${AUTH_JWT_SECRET}          # env 에서 fetch
```

---

### 2.4 refresh token raw

**왜 안 됨**
- raw 도난 = 14일 사용.
- DB 에 hash 만 저장.

**구현** — [[../database/refresh-tokens-table#2.3]] 의 SHA-256 hash.

---

## 3. 마스킹 정책 — PII

### 3.1 Email 마스킹

```java
public static String maskEmail(String email) {
    int at = email.indexOf('@');
    if (at < 2) return "***@***";
    return email.charAt(0) + "***" + email.substring(at - 1);
    // "alice@example.com" → "a***e@example.com"
}
```

**왜 부분 마스킹 (전체 X 아님)**
- 디버깅 / CS 응대 시 "어떤 user 인지" 식별 필요.
- 평문 전체는 불필요 (PII 보호).

### 3.2 Phone 마스킹

```java
public static String maskPhone(String phone) {
    // "01012345678" → "010-****-5678"
    if (phone == null || phone.length() < 8) return "***";
    return phone.substring(0, 3) + "-****-" + phone.substring(phone.length() - 4);
}
```

### 3.3 Name 마스킹

```java
public static String maskName(String name) {
    // "홍길동" → "홍*동"
    if (name.length() <= 1) return "*";
    if (name.length() == 2) return name.charAt(0) + "*";
    return name.charAt(0) + "*".repeat(name.length() - 2) + name.charAt(name.length() - 1);
}
```

---

## 4. Hibernate SQL 로그 정책

```yaml
# application.yml (prod)
logging:
  level:
    org.hibernate.SQL: WARN              # INFO 이상 — SQL 안 찍음
    org.hibernate.type.descriptor.sql: WARN
```

**왜 prod 에선 WARN**
- INFO / DEBUG 시 SQL 의 parameter (password / phone) 그대로 노출.
- 한국 사례: Hibernate SQL 로그 유출로 PII 사고.

**dev 환경**
- DEBUG OK (디버깅 필요).
- 단 prod 와 다른 DB / 가짜 데이터.

---

## 5. 응답 DTO 정책

```java
// ❌ Entity 그대로 반환
@GetMapping("/me")
public UserJpaEntity me(...) { return user; }       // password_hash 노출!

// ✅ DTO 명시
@GetMapping("/me")
public UserResponse me(...) {
    return new UserResponse(user.id(), user.email(), user.name(), user.status());
    // password_hash / phone_hash 명시적으로 제외
}
```

**왜 명시적 DTO**
- Entity 자동 직렬화 = 모든 컬럼 노출.
- DTO = 필요한 필드만.

**원칙**
- response 는 항상 record 로.
- 필드를 명시 작성.

---

## 6. 로그에 들어가야 하는 / 안 되는

### 6.1 OK

- user_id (ULID — 추측 불가)
- correlation_id / trace_id
- HTTP status / method / path
- timing / latency
- 에러 type (Exception class name)

### 6.2 마스킹 후 OK

- email (앞 1 + ... + 뒷 1)
- phone (앞 3 + 끝 4)
- name (앞 1 + ... + 끝 1)
- IP address (옵션 — GDPR 보유 기간 정책)

### 6.3 절대 X

- 평문 password
- password_hash
- JWT secret
- raw refresh token
- raw verification code (6-digit)
- 카드번호 / 주민번호 (본 vault 외 영역)

---

## 7. Sensitive Header

```java
@RestController
public class FooController {
    @PostMapping("/foo")
    public Response foo(
        @RequestHeader(name = "Authorization") String auth,    // ❌ 로그에 X
        @RequestBody FooRequest req
    ) {
        log.info("foo request from user: {}", req);             // toString 마스킹 보장
    }
}
```

**왜 Authorization 헤더를 로그에 X**
- Bearer 뒤의 JWT 가 access token.
- 로그 유출 시 즉시 사용 가능 (15분 안).

**구현**
- log 의 헤더 출력 시 Authorization 필터링.
- AccessLog 의 customizer.

```java
// Sentry / structured logging
log.info("request", kv("method", "POST"), kv("path", "/foo"), kv("user", maskEmail(...)));
// Authorization 자동 제외
```

---

## 8. ObjectMapper / Jackson 설정

```java
@Bean
public ObjectMapper objectMapper() {
    var mapper = new ObjectMapper();
    mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);   // null 제외
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    return mapper;
}
```

**왜 NON_NULL**
- password / password_hash 필드가 null 인 경우 응답에 안 나타남.
- 의도하지 않은 노출 차단.

---

## 9. 함정 모음

### 함정 1 — record 의 기본 toString 으로 password 노출
`log.info("req: {}", request)` 가 평문 password 로그.
→ password 가 있는 record 는 toString 오버라이드.

### 함정 2 — Entity 그대로 응답
모든 컬럼 노출 (password_hash 포함).
→ DTO 명시.

### 함정 3 — Hibernate SQL 로그 prod 활성
SQL parameter 가 password / phone 평문.
→ prod = WARN.

### 함정 4 — `@JsonIgnore` 누락 (password_hash 컬럼)
직렬화 시 노출.
→ 모든 sensitive 컬럼 명시.

### 함정 5 — log 의 Authorization 헤더
JWT 가 그대로 → 도난 시 15분 사용.
→ filter / customizer 제외.

### 함정 6 — 응답에 internal field
debug 목적 추가 후 prod 에 잔존.
→ 명시 DTO + code review.

### 함정 7 — JWT secret 코드 hardcoded
git history 에 영구 노출.
→ env / KMS.

### 함정 8 — Stack trace 응답
Internal class / path 노출 = 공격자 reconnaissance.
→ `@ControllerAdvice` 가 production message 만 반환.

### 함정 9 — 6-digit code 의 raw 로그
"인증코드 발송: 123456" → 로그 유출 시 도용.
→ hash 만 로그.

### 함정 10 — Sentry 에 평문 PII
Sentry breadcrumbs / context.
→ scrubbing rules 활성.

### 함정 11 — Heap dump 에 평문
JVM heap dump (OOM 시) 가 메모리 안의 password / JWT 포함.
→ prod heap dump 제한 / 빠른 폐기.

### 함정 12 — Marketing tools (Datadog / GA) 에 PII
사용자 분석 도구가 PII 수집 → GDPR 위반.
→ 익명 user_id 만 (email / phone 제외).

---

## 10. 관련

- [[security|↑ security hub]]
- [[../domain-model/password-hash-vo]] — PasswordHash VO + 마스킹
- [[../domain-model/email-vo]] · [[../domain-model/phone-number-vo]]
- [[../database/encryption-at-rest]] — DB 레벨 보호
- [[../../pitfalls/null-safety]] — 평문 password 의 함정
- 외부 — OWASP Logging Cheat Sheet, GDPR PII 정의
