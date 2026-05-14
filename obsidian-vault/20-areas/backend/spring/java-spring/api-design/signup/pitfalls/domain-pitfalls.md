---
title: "도메인 모델 / 검증 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - pitfalls
  - domain
  - validation
---

# 도메인 모델 / 검증 함정

**[[pitfalls|↑ pitfalls hub]]**

> DDD 적용 시 흔한 실수 + Bean Validation 의 함정.

---

## 함정 1 — 도메인이 `@Entity` 와 결합

### 무엇

```java
@Entity
@Table(name = "users")
public class User {
    @Id String id;
    @Column String email;

    public void verifyEmail() { ... }    // 도메인 로직
}
```

### 왜 위험

- ORM 변경 시 도메인 재작성.
- Hibernate lazy proxy 가 도메인 객체에 누수.
- 도메인 객체가 DB 의존 — 단위 테스트 어려움.
- 도메인 invariant 가 JPA 의 default constructor / setter 와 충돌.

### 해결

- 도메인 `User` (pure Java) ↔ JPA `UserJpaEntity` 분리.
- Adapter 가 매핑.

```java
// Domain
public final class User { /* pure java */ }

// Infrastructure (JPA)
@Entity public class UserJpaEntity { /* ... */ }

// Adapter
public class UserRepositoryImpl implements UserRepository {
    UserJpaEntity entity = jpaRepo.findById(id);
    return entity.toDomain();
}
```

자세히: [[../domain-model/user-aggregate]].

---

## 함정 2 — Setter 노출

### 무엇

```java
public class User {
    @Setter private UserStatus status;          // ❌
}

user.setStatus(UserStatus.ACTIVE);              // 어디서든 호출 가능
```

### 왜 위험

- 상태 머신 무의미.
- 도메인 invariant (PENDING → ACTIVE 만 OK) 보장 X.
- 어디서 status 변경됐는지 추적 어려움.

### 해결

```java
public class User {
    private UserStatus status;

    public void verifyEmail() {
        if (status != PENDING_VERIFICATION)
            throw new IllegalStateException();
        status = ACTIVE;
        addEvent(new UserVerified(id));
    }

    public void suspend(String reason) {
        if (status != ACTIVE) throw ...;
        status = SUSPENDED;
        addEvent(new UserSuspended(id, reason));
    }
}
```

→ 의미 있는 메서드만. setter X.

---

## 함정 3 — Aggregate 가 너무 큼

### 무엇

```java
public class User {
    List<Order> orders;
    List<Post> posts;
    List<Follower> followers;
    Profile profile;
    // ...
}
```

### 왜 위험

- 단일 트랜잭션에 너무 많은 entity load.
- N+1 / 메모리 OOM.
- User 변경 시 모든 관련 entity dirty check.

### 해결

- Aggregate 경계 좁게.
- 다른 도메인 (Order, Post) 는 ID 참조 + 별도 Aggregate.

```java
public class User {
    // 자기 정보만
}

public class Order {
    UserId buyerId;      // 참조
}
```

자세히: [[../domain-model/aggregate-boundaries]].

---

## 함정 4 — Domain Event 없음

### 무엇

```java
@Transactional
public User signup(...) {
    var user = users.save(...);
    emailService.sendVerification(user);
    smsService.notify(admin);
    analyticsService.track(...);
    return user;
}
```

### 왜 위험

- UseCase 가 모든 외부 시스템 의존.
- 추가 후속 처리 시 UseCase 수정.
- 트랜잭션 안에 외부 호출 (함정).

### 해결

```java
@Transactional
public User signup(...) {
    var user = users.save(...);
    eventPublisher.publishEvent(new UserRegistered(user.id()));
    return user;
}

@Component
class EmailNotificationListener {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onRegistered(UserRegistered e) {
        emailService.sendVerification(...);
    }
}
```

→ UseCase 가 외부 시스템 모름. 새 후속 처리 = 새 listener.

---

## 함정 5 — Value Object 검증 누락

### 무엇

```java
public record Email(String value) { }   // 검증 없음
```

```java
new Email("not-an-email")               // 통과 → DB 에 garbage
```

### 해결

```java
public record Email(String value) {
    private static final Pattern PATTERN = Pattern.compile(...);

    public Email {
        if (value == null || !PATTERN.matcher(value).matches())
            throw new IllegalArgumentException("invalid email");
    }
}
```

자세히: [[../domain-model/email-vo]].

---

## 함정 6 — VO 가 mutable

### 무엇

```java
public class Email {
    private String value;
    public void setValue(String v) { this.value = v; }    // ❌
}
```

### 왜 위험

- VO 본질 위반 — equals/hashCode 안정성 X.
- HashSet / HashMap 이상 동작.

### 해결

- record 사용 — 자동 final + immutable.
- 또는 final class + final field.

---

## 함정 7 — VO 가 자동 정규화

### 무엇

```java
public record Email(String value) {
    public Email {
        value = value.trim().toLowerCase();    // ❌ 자동 변환
    }
}
```

### 왜 위험

- 사용자가 모르게 input mutation.
- 디버깅 시 input 추적 어려움.

### 해결

```java
// VO 는 받은 값 그대로 보존 + 검증만
public record Email(String value) {
    public Email {
        if (!valid(value)) throw ...;
    }
}

// 호출자 (Service) 가 명시적 정규화
var email = new Email(input.trim().toLowerCase(Locale.ROOT));
```

자세히: [[../domain-model/email-vo#4 정규화]].

---

## 함정 8 — `String.toLowerCase()` (Locale 없이)

### 무엇

```java
email.toLowerCase()
```

### 왜 위험

- 터키어 환경 — `'I'` → `'ı'` (점 없는 i).
- 그리스어도 비슷 issue.
- 같은 이메일이 다른 lowercase 결과.

### 해결

```java
email.toLowerCase(Locale.ROOT)
```

명시.

---

## 함정 9 — Bean Validation 없이 컨트롤러 진입

### 무엇

```java
@PostMapping("/signup")
public Response signup(@RequestBody SignupRequest req) {     // @Valid 없음
    ...
}
```

### 왜 위험

- null body / 빈 필드가 service 까지.
- 도메인 VO 에서 throw → 422 가 아닌 500.

### 해결

```java
@PostMapping("/signup")
public Response signup(@Valid @RequestBody SignupRequest req) {
    ...
}
```

모든 `@RequestBody` 에 `@Valid`.

---

## 함정 10 — `@AssertTrue` 메시지 없음

### 무엇

```java
public record SignupRequest(
    @AssertTrue Boolean termsAgreed,
    ...
) { }
```

### 왜 위험

- termsAgreed=false 시 응답 — "must be true" (default).
- 사용자 메시지 의미 약함.

### 해결

```java
@AssertTrue(message = "약관에 동의해주세요")
Boolean termsAgreed
```

또는 ResponseCode 매핑 — `ApiExceptionHandler`.

---

## 함정 11 — Email / Name 의 trim / 정규화 누락

### 무엇

```java
public Response signup(@RequestBody SignupRequest req) {
    users.save(new User(req.email(), ...));   // "  Alice@X.COM  " 그대로
}
```

### 해결

```java
public Response signup(...) {
    var normalized = req.email().trim().toLowerCase(Locale.ROOT);
    users.save(new User(normalized, ...));
}
```

→ Application service 가 정규화.

---

## 함정 12 — 한국어 / Unicode name 거절

### 무엇

```java
@Pattern(regexp = "[a-zA-Z]+")
String name
```

### 왜 위험

- 한국 사용자 가입 불가.

### 해결

- `@Size(min=1, max=100)` 만.
- regex 검증 시 unicode-aware:
  ```java
  @Pattern(regexp = "[\\p{L}\\s]+")    // 모든 letter
  ```

---

## 함정 13 — Aggregate root 가 internal entity 노출

### 무엇

```java
public class User {
    public List<UserTermsConsent> getConsents() { return consents; }    // ❌
}

user.getConsents().add(new UserTermsConsent(...));    // 외부에서 직접 mutation
```

### 해결

```java
public class User {
    public void agreeToTerms(Terms terms, Instant now) { ... }
    public boolean hasAgreedTo(Terms terms) { ... }
}
```

→ 메서드 통해서만.

---

## 함정 14 — Aggregate 간 직접 객체 참조

### 무엇

```java
public class Order {
    User buyer;          // ❌ 다른 Aggregate 의 root 객체 참조
}
```

### 왜 위험

- 트랜잭션 경계 깨짐.
- N+1 / cascade 위험.

### 해결

```java
public class Order {
    UserId buyerId;      // ID 만
}
```

자세히: [[../domain-model/aggregate-boundaries#6 ID 참조]].

---

## 함정 15 — 도메인이 외부 호출

### 무엇

```java
public class User {
    public void verifyEmail() {
        smsService.send(phone, "...");    // ❌ 도메인이 외부 의존
        status = ACTIVE;
    }
}
```

### 해결

- 도메인 = pure logic + event 발행만.
- listener 가 외부 호출.

```java
public class User {
    public void verifyEmail() {
        status = ACTIVE;
        addEvent(new UserVerified(id));
    }
}
```

---

## 함정 16 — Domain 코드에 framework annotation

### 무엇

```java
public class User {
    @Autowired SomeService service;    // ❌
}
```

### 왜 위험

- 도메인 ↔ framework 결합.
- 단위 테스트 어려움.

### 해결

- 도메인 = pure Java.
- 의존성은 Application service 가 주입.

---

## 함정 17 — DTO 와 도메인 객체 혼용

### 무엇

```java
@PostMapping("/signup")
public User signup(@RequestBody User user) {    // ❌ 도메인 객체 그대로
    ...
}
```

### 왜 위험

- API 변경이 도메인 변경 (반대도).
- 응답에 도메인 internal 노출.

### 해결

- DTO (`SignupRequest`, `UserResponse`) 별도.
- Controller / Application service 에서 매핑.

---

## 관련

- [[pitfalls|↑ pitfalls hub]]
- [[../domain-model]] — 도메인 모델 전체
- [[../domain-model/user-aggregate]] · [[../domain-model/value-objects]] · [[../domain-model/aggregate-boundaries]]
