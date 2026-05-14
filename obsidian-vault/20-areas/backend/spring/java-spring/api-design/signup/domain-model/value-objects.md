---
title: "Value Objects — 패턴 + 4종 소개"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - domain-model
  - value-object
---

# Value Objects — 패턴 + 4종 소개

**[[domain-model|↑ domain-model hub]]**

> 도메인 layer 의 가장 기본 단위. **불변 + 자체 검증** 으로 invalid 한 객체가 도메인에 못 들어오게.

---

## 1. Value Object 란

| 특성 | 설명 |
| --- | --- |
| **불변 (Immutable)** | 한 번 생성되면 안 바뀜. 변경 = 새 인스턴스. |
| **자체 검증** | 생성 시점에 invalid 면 예외 → 도메인 안엔 항상 valid 한 값만 |
| **identity 없음** | 같은 값이면 같은 객체 (`equals` / `hashCode` 자동) |
| **side effect 없음** | DB / 외부 호출 X |
| **scope 좁음** | 단일 책임 — 한 가지 도메인 개념 |

→ Entity (id 로 식별) 와 대비. VO 는 **값 자체** 가 identity.

---

## 2. 왜 `record` 인가 (Java 14+)

```java
public record Email(String value) {
    public Email {
        if (value == null || !VALID.matcher(value).matches())
            throw new IllegalArgumentException("invalid email");
    }
}
```

vs 기존 class:

```java
public final class Email {
    private final String value;
    public Email(String value) {
        if (value == null || !VALID.matcher(value).matches())
            throw new IllegalArgumentException(...);
        this.value = value;
    }
    public String value() { return value; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}
```

**record 의 이득**:
- 보일러플레이트 0 — `equals`/`hashCode`/`toString`/getter 자동
- **final** + **immutable** 자동 (필드 final + 메서드 noir)
- **compact constructor** — 검증 자리가 명확
- Sealed / pattern matching 친화

**record 의 한계**:
- 모든 필드가 public (getter 노출)
- 상속 X
- 직렬화 시 Jackson 호환은 OK (constructor 사용)

→ Value Object 는 **record 가 정답**.

---

## 3. Compact Constructor — 검증의 자리

```java
public record Email(String value) {
    public Email {                                            // ← compact constructor
        if (value == null)
            throw new IllegalArgumentException("email required");
        if (!EMAIL_PATTERN.matcher(value).matches())
            throw new IllegalArgumentException("invalid email format");
        if (value.length() > 254)
            throw new IllegalArgumentException("email too long (RFC 5321: 254)");
    }
}
```

> 💡 **왜 `IllegalArgumentException`?**
> 도메인 규칙 위반은 **프로그래밍 오류** (불변식). business exception (RuntimeException 의 자식) 아님.
> Adapter / Controller 가 user input → VO 변환 시점에 catch + 422 응답으로 변환 ([[../../common/response-envelope#6]]).

> 💡 **왜 normalization 안 함?**
> `Email("Alice@X.com")` 이 `alice@x.com` 으로 자동 변환? — **반대 권장**.
> VO 는 **받은 값 그대로 보존**. 정규화는 **호출자 (Application Service)** 가:
> ```java
> var email = new Email(input.trim().toLowerCase(Locale.ROOT));
> ```
> 이유: VO 가 input mutation = 디버깅 어려움. caller 가 input 흐름 명확히 알기.

---

## 4. equals / hashCode — 값 동등성

```java
new Email("alice@x.com").equals(new Email("alice@x.com"))    // true (값 동등)
new Email("alice@x.com") == new Email("alice@x.com")          // false (다른 instance)
```

→ record 가 자동 처리. `Set<Email>` / `Map<Email, ...>` 에 자연스럽게.

---

## 5. 이 폴더의 VO 4종 (+ 5번째)

| VO | 노트 | 역할 |
| --- | --- | --- |
| [[email-vo]] | Email | RFC 5322 검증 |
| [[password-hash-vo]] | PasswordHash | argon2 PHC 형식만 |
| [[user-id-vo]] | UserId | ULID (26 char) |
| [[phone-number-vo]] | PhoneNumber | 한국 휴대폰 / 정규화 |
| RefreshTokenId / VerificationTokenId | (UserId 와 같은 패턴) | ULID identifier |

각 VO 의 깊이 — 자기 노트.

---

## 6. JPA 매핑 — VO 가 DB 컬럼으로

Adapter 가 매핑 책임:

```java
@Entity
public class UserJpaEntity {
    @Id @Column(length = 26) private String id;              // UserId.value()
    @Column(nullable = false, length = 254) private String email;     // Email.value()
    @Column(name = "password_hash", length = 255) private String passwordHash;
    // ...

    // Adapter 가 매핑
    public User toDomain() {
        return User.reconstitute(
            new UserId(id),
            new Email(email),
            phone != null ? new PhoneNumber(phone) : null,
            new PasswordHash(passwordHash),
            ...
        );
    }
}
```

> 💡 **왜 `@Embeddable` 안 씀?**
> JPA `@Embeddable + @Embedded` 로 VO 매핑 가능. 단 record 는 `@Embeddable` 호환 X (constructor 문제). final class 로 가야 함.
> 본 vault: **명시적 String 컬럼 + Adapter 매핑** — record 그대로 사용.

---

## 7. VO 의 행동 (methods) — 어디까지 OK

VO 는 "값" 이지만 행동도 가질 수 있음:

```java
public record Money(long amountMinorUnit, Currency currency) {
    public Money add(Money other) {
        ensureSame(other);
        return new Money(amountMinorUnit + other.amountMinorUnit, currency);
    }
    public Money multiply(int n) {
        return new Money(amountMinorUnit * n, currency);
    }
}
```

→ **불변 — 새 instance 반환**. 원본 변경 X.

규칙:
- ✅ 같은 VO 내의 연산 (`add`, `multiply`)
- ✅ format / serialize
- ❌ 다른 Aggregate / Repository 호출
- ❌ side effect

---

## 8. 공통 함정

### 함정 1 — VO 가 mutable
`Email.setValue(...)` 같은 setter = VO 본질 위반. **field final + setter 없음**.

### 함정 2 — 자체 검증 X
Bean Validation 만 의존 (`@Email`) → 도메인 layer 안에서도 invalid 한 Email 가능.
**compact constructor + Bean Validation 둘 다** (이중 방어).

### 함정 3 — VO 가 null
`new Email(null)` 통과? compact constructor 에서 명시 거절.

### 함정 4 — Hash code 안 됨 (구식 class + equals 만 override)
HashSet / HashMap 에서 이상 동작. **둘 다 override 또는 record**.

### 함정 5 — Locale 의존
`"Alice@X.com".toLowerCase()` — Locale 따라 다를 수 있음 (터키어 'I' 문제).
**`toLowerCase(Locale.ROOT)`** 명시.

### 함정 6 — 너무 큰 VO
`Money(amount, currency, tax, discount, ...)` = 사실상 Entity.
VO 는 **단일 개념**. 분리.

### 함정 7 — VO 가 Domain Event 발행
VO 는 행동 / 이벤트 X. **Aggregate 가 발행**.

### 함정 8 — JPA `@Embeddable` 의 record 호환성
record + `@Embeddable` 은 JPA 3.x 에서도 까다로움. **String 컬럼 + Adapter 매핑** 권장.

---

## 9. 관련

- [[domain-model|↑ domain-model hub]]
- [[email-vo]] · [[password-hash-vo]] · [[user-id-vo]] · [[phone-number-vo]]
- [[user-aggregate]] — VO 를 활용하는 Aggregate
- [[../../pitfalls/null-safety]] — record + compact constructor 의 null 방어
