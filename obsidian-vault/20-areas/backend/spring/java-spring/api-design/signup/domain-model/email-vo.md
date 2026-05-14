---
title: "Email VO — RFC 5322 + 정규화 위치"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:07:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - value-object
  - email
---

# Email VO — RFC 5322 + 정규화 위치

**[[domain-model|↑ domain-model hub]]**  ·  ← [[value-objects]]

---

## 1. 코드

```java
// src/main/java/com/example/shop/domain/user/Email.java
public record Email(String value) {

    private static final Pattern EMAIL = Pattern.compile(
        "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"
    );

    public Email {
        if (value == null)
            throw new IllegalArgumentException("email required");
        if (value.length() > 254)
            throw new IllegalArgumentException("email too long (max 254)");
        if (!EMAIL.matcher(value).matches())
            throw new IllegalArgumentException("invalid email format: " + value);
    }
}
```

---

## 2. 왜 이 regex — RFC 5322 vs 단순화

### 2.1 진짜 RFC 5322 의 복잡함

RFC 5322 / 6531 의 완전한 regex 는 **3000자 이상**. quote / 주석 / international 까지 다 받음. 실용성 X.

예 (RFC 완전판 일부):
```
[+-]?(\d+\.?\d*|\.\d+)([eE][+-]?\d+)?...
```

### 2.2 본 vault 의 단순화 regex

```regex
^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$
```

| 부분 | 의미 |
| --- | --- |
| `^[A-Za-z0-9._%+-]+` | local part (영문/숫자/`.`/`_`/`%`/`+`/`-`) |
| `@` | 구분자 |
| `[A-Za-z0-9.-]+` | domain |
| `\.[A-Za-z]{2,}$` | TLD (최소 2 char) |

**커버리지**:
- ✅ `alice@example.com` / `bob.smith+filter@gmail.com`
- ✅ 한글 도메인 IDN — Punycode 변환 후 (`@xn--example.kr` 식)
- ❌ quote 포함 (`"weird email"@x.com`) — 실세계 거의 없음
- ❌ IPv6 literal (`@[::1]`) — 사용 X

→ 99.9% 실세계 이메일 통과 + 안전.

### 2.3 더 엄격하게 — `Apache Commons EmailValidator`

```java
import org.apache.commons.validator.routines.EmailValidator;

public Email {
    if (!EmailValidator.getInstance().isValid(value))
        throw new IllegalArgumentException(...);
}
```

장점: RFC 5322 더 충실
단점: 외부 의존성 + 통상 우리 regex 와 결과 거의 같음

본 vault: **단순 regex** + Bean Validation `@Email` 이 추가 검증 (Controller 단).

---

## 3. 254 char 길이 제한

RFC 5321 의 **maximum length 254** (local 64 + `@` + domain 255 - 1 = 256, 단 leading dot 등 제외 = 254 실용).

```java
if (value.length() > 254)
    throw new IllegalArgumentException("email too long");
```

→ DB `VARCHAR(254)` 일치.

---

## 4. 정규화 — 어디서 하나

### 4.1 본 vault 정책: **Application Service 가 정규화, VO 는 그대로**

```java
// Application
public User signup(SignupCommand cmd) {
    var email = new Email(cmd.email().trim().toLowerCase(Locale.ROOT));
    //                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                            여기서 정규화
    if (users.existsByEmail(email)) throw ...
    ...
}
```

```java
// Email VO — 받은 값 그대로
public record Email(String value) {
    public Email {
        // 형식 검증만, 정규화 X
        if (!EMAIL.matcher(value).matches()) ...
    }
}
```

### 4.2 왜 VO 가 정규화 안 함

| VO 가 정규화 시 | VO 가 안 할 시 (본 vault) |
| --- | --- |
| `new Email("Alice@X.com").value()` → `"alice@x.com"` (자동 변환) | 그대로 `"Alice@X.com"` |
| 사용자가 모르게 input 변환 | caller 가 명시적 정규화 |
| 디버깅 시 input 추적 어려움 | input 흐름 명확 |
| 다른 정규화 정책 필요 시 (e.g. case-preserve) 코드 변경 | caller 가 정책 결정 |

→ **VO 는 "검증 + 보존"**, Service 가 정규화. 단일 책임.

### 4.3 case-insensitive UNIQUE — DB 에서 처리

```sql
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));
```

→ DB 가 case-insensitive 보장. 우리는 lowercase 로 저장 (Service 가 변환) + DB UNIQUE 가 안전망.

자세히: [[../database/users-table]].

---

## 5. Bean Validation 과의 관계

```java
// Controller DTO
public record SignupRequest(
    @Email @NotBlank @Size(max = 254) String email,
    ...
)
```

| Layer | 책임 |
| --- | --- |
| Controller (Bean Validation) | 1차 — `@Email` + `@Size` |
| Domain VO (record + compact constructor) | 2차 — 형식 + 길이 |
| DB constraint | 3차 — UNIQUE / length |

→ **3중 방어**. 어디서든 한 곳 빠져도 다른 곳이 잡음.

> 💡 **Bean `@Email`** 은 도메인 VO 보다 관대 (예: `a@b` 통과). 도메인 VO 가 더 엄격하게.

---

## 6. 응답 / 직렬화

```java
var json = objectMapper.writeValueAsString(new Email("alice@x.com"));
// 결과: "alice@x.com"  (record accessor 사용)
```

→ Jackson 이 record 의 single-field accessor 를 자동 활용. 추가 설정 없이 String 으로 직렬화.

만약 wrapper 형태로:
```json
{ "email": { "value": "alice@x.com" } }
```
가 싫으면 — `@JsonValue` 추가:

```java
public record Email(@JsonValue String value) { ... }
```

→ `"alice@x.com"` 단순 String 으로.

본 vault 기본: **`@JsonValue` 사용** — API 응답 깔끔.

---

## 7. PII / 로깅

이메일은 PII (개인정보). 로그 출력 시:

```java
// ❌ 전체 노출
log.info("user email: {}", email.value());

// ✅ 마스킹
log.info("user email: {}", mask(email.value()));

private static String mask(String email) {
    int at = email.indexOf('@');
    if (at < 2) return "***@***";
    return email.charAt(0) + "***" + email.substring(at - 1);
    // "alice@x.com" → "a***e@x.com"
}
```

→ DEBUG 외엔 마스킹. PII 보호 정책.

---

## 8. 함정 모음

### 함정 1 — `@Email` 만 의존
Bean Validation `@Email` 이 RFC 5322 너무 관대 — `a@b` 도 통과.
**도메인 VO 가 더 엄격하게** 이중 방어.

### 함정 2 — VO 가 정규화 (자동 lowercase)
사용자가 모르게 변환 = 디버깅 어려움. **Service 가 명시적 변환**.

### 함정 3 — `String.toLowerCase()` (Locale 없이)
터키어 / 그리스어 환경에서 'I' → 'ı' 변환 (특수). **`toLowerCase(Locale.ROOT)`** 명시.

### 함정 4 — 254 char 제한 없음
악의적 입력 (10000 char email) → DB INSERT 실패 / 성능 저하. **VO 에서 차단**.

### 함정 5 — 한글 / 유니코드 이메일
RFC 6531 — `홍길동@서울시.kr` 가능. 본 vault regex 는 ASCII 만. 필요시 IDN 라이브러리.

### 함정 6 — IDN 도메인 (Punycode)
`@한글도메인.kr` → `@xn--...kr` 변환. 일관성 위해 **모두 Punycode 변환 후 저장** (Application 단에서).

### 함정 7 — `+` alias
`alice+filter@gmail.com` 같은 plus alias. Gmail 은 `alice@gmail.com` 과 같은 inbox.
사용자가 alias 로 가입 시 - 정책 결정 (허용 / 차단 / canonicalize).
본 vault: **alias 허용** (Gmail 의 의도된 기능).

### 함정 8 — 응답에 raw 이메일
PII 노출. Internal 만 raw, 외부 응답엔 마스킹 옵션 검토.

---

## 9. 관련

- [[domain-model|↑ domain-model hub]]
- [[value-objects]]
- [[../enums/social-provider-type]] — Apple 의 email 부재 케이스
- [[../database/users-table]] — `lower(email)` UNIQUE
- [[../security#3 민감정보 처리]] — PII 로깅
