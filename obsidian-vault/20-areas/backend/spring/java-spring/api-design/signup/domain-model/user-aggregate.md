---
title: "User Aggregate — Root + 메서드 + transition"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:02:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - domain-model
  - aggregate
---

# User Aggregate — Root + 메서드 + transition

**[[domain-model|↑ domain-model hub]]**

> auth 도메인의 핵심 Aggregate Root. 모든 user 상태 변경의 게이트.

---

## 1. 전체 코드

```java
// src/main/java/com/example/shop/domain/user/User.java
public final class User {

    private final UserId id;
    private final Email email;
    private PhoneNumber phone;                   // 휴대폰 인증 사용 시
    private PasswordHash passwordHash;
    private final String name;
    private UserStatus status;
    private Role role;
    private final SocialProviderType providerType;
    private final String externalId;             // 소셜 가입의 provider 고유 ID (null=LOCAL)
    private Instant emailVerifiedAt;
    private Instant phoneVerifiedAt;
    private final Instant createdAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private User(UserId id, Email email, PhoneNumber phone, PasswordHash hash, String name,
                 UserStatus status, Role role, SocialProviderType providerType, String externalId,
                 Instant emailVerifiedAt, Instant phoneVerifiedAt, Instant createdAt) {
        this.id = id; this.email = email; this.phone = phone; this.passwordHash = hash;
        this.name = name; this.status = status; this.role = role;
        this.providerType = providerType; this.externalId = externalId;
        this.emailVerifiedAt = emailVerifiedAt; this.phoneVerifiedAt = phoneVerifiedAt;
        this.createdAt = createdAt;
    }
}
```

> 💡 **왜 `final class`?**
> Aggregate Root 는 상속 X. 도메인 규칙이 subclass 에서 깨질 위험 차단.

> 💡 **왜 private 생성자?**
> 외부에서 직접 `new User(...)` 못 만듦. 반드시 static factory (`register` / `reconstitute`) 거침.
> → 모든 instantiation 이 검증된 경로 1개로 흐름.

> 💡 **왜 `events` 가 ArrayList?**
> 동시성 우려가 있을 수 있지만 — Aggregate 는 **한 트랜잭션 안에서 단일 스레드** 사용 가정.
> 그래서 ArrayList OK. ConcurrentLinkedQueue 까진 과함.

---

## 2. Static Factory — `register(...)` — 가입

```java
public static User register(UserId id, Email email, PasswordHash hash,
                            String name, Role role, SocialProviderType providerType,
                            String externalId, Instant now) {
    validateName(name);
    var u = new User(id, email, null, hash, name,
                     UserStatus.PENDING_VERIFICATION, role,
                     providerType, externalId,
                     null, null, now);
    u.events.add(new UserRegistered(id, email, providerType, now));
    return u;
}

/** 소셜 가입 — 이메일 인증 자동 처리. */
public static User registerSocial(UserId id, Email email, String name,
                                   SocialProviderType provider, String externalId,
                                   Instant now) {
    validateName(name);
    var u = new User(id, email, null, null, name,
                     UserStatus.ACTIVE,                    // 소셜은 바로 ACTIVE
                     Role.USER, provider, externalId,
                     now, null, now);                       // emailVerifiedAt 즉시
    u.events.add(new UserRegistered(id, email, provider, now));
    u.events.add(new UserEmailVerified(id, email, now));    // 자동 verified event
    return u;
}

private static void validateName(String name) {
    if (name == null || name.isBlank() || name.length() > 100)
        throw new IllegalArgumentException("invalid name: 1~100 chars required");
}
```

> 💡 **왜 `register` 가 static factory?**
> 1. 생성과 동시에 `UserRegistered` event 발행 — 생성자에선 어색
> 2. 이름 명확 (`new User(...)` 보다 `User.register(...)` 가 의도 분명)
> 3. 다른 factory (`registerSocial`, `reconstitute`) 와 의미 구분

> 💡 **왜 소셜은 ACTIVE?**
> Apple / Google / Kakao 가 이미 email 검증함 — 우리가 또 메일 보낼 필요 X.
> 단, `email_verified=false` 응답인 Google 케이스는 거절 (provider 검증 단에서).

> 💡 **왜 password 가 nullable?**
> 소셜 user 는 password 없음. LOCAL 만 `passwordHash NOT NULL`.
> → DB 도 `password_hash` nullable. ([[../database/users-table|database]] 참고)

---

## 3. Transition — `verifyEmail()`

```java
public void verifyEmail(Instant now) {
    if (status == UserStatus.DELETED)
        throw new IllegalStateException("deleted user");
    if (emailVerifiedAt != null) return;                    // 멱등 — 이미 인증
    if (status == UserStatus.PENDING_VERIFICATION) {
        status = UserStatus.ACTIVE;                          // PENDING → ACTIVE 전이
    }
    emailVerifiedAt = now;
    events.add(new UserEmailVerified(id, email, now));
}
```

> 💡 **왜 멱등 (이미 인증이면 no-op)?**
> 이메일 인증 메일 link 를 사용자가 두 번 클릭 = 흔함.
> 두 번째 호출이 예외 던지면 UX 나쁨. **return** 으로 silent.

> 💡 **왜 PENDING 일 때만 ACTIVE 로?**
> 이미 ACTIVE 인 user 가 이메일 재인증 (e.g. 이메일 변경 후 재검증) — status 는 ACTIVE 유지.
> SUSPENDED 인 user 가 이메일 인증 = status 유지 (정지 해제는 별도 흐름).

> 💡 **왜 `now` 가 인자?**
> 도메인은 `Clock` 모름. testability — fixed clock 으로 테스트 가능.

---

## 4. Transition — `verifyPhone()`

```java
public void verifyPhone(PhoneNumber phone, Instant now) {
    if (status == UserStatus.DELETED)
        throw new IllegalStateException("deleted user");
    if (this.phone != null && this.phone.equals(phone) && phoneVerifiedAt != null)
        return;                                              // 같은 번호 이미 인증
    this.phone = phone;
    this.phoneVerifiedAt = now;
    events.add(new UserPhoneVerified(id, phone, now));
}
```

> 💡 **왜 phone 을 여기서 set?**
> 가입 시점엔 phone null. 추후 휴대폰 인증 통과 시 set + verifiedAt 같이.
> "휴대폰 인증" 이라는 transition 의 책임이 도메인에 있음.

---

## 5. Transition — `changePassword()`

```java
public void changePassword(PasswordHash newHash, Instant now) {
    if (status == UserStatus.DELETED)
        throw new IllegalStateException("deleted user");
    if (providerType != SocialProviderType.LOCAL)
        throw new IllegalStateException("social account cannot change password");
    this.passwordHash = newHash;
    events.add(new UserPasswordChanged(id, now));
}
```

> 💡 **왜 소셜 user 의 changePassword 거절?**
> 소셜 user 는 password 자체가 null. 변경 = 의미 없음.
> 만약 LOCAL 로 전환 의도면 별도 흐름 (`linkLocalCredentials(password)`).

> 💡 **왜 `PasswordHash` (이미 해시) 받음?**
> 도메인이 평문 password 를 받지 X. **Application layer 의 PasswordEncoder 가 해시 후 전달**.
> 도메인은 hash 만 핸들링 — 평문 누수 위험 최소.

---

## 6. Transition — `suspend()` / `unsuspend()` / `delete()`

```java
public void suspend(Instant now) {
    if (status != UserStatus.ACTIVE)
        throw new IllegalStateException("only ACTIVE can be suspended");
    status = UserStatus.SUSPENDED;
    events.add(new UserSuspended(id, now));
}

public void unsuspend(Instant now) {
    if (status != UserStatus.SUSPENDED)
        throw new IllegalStateException("not suspended");
    status = UserStatus.ACTIVE;
    events.add(new UserUnsuspended(id, now));
}

public void delete(Instant now) {
    if (status == UserStatus.DELETED) return;                // 멱등
    status = UserStatus.DELETED;
    events.add(new UserDeleted(id, now));
}
```

> 💡 **왜 delete 가 멱등?**
> 동시 delete 요청 / retry — DELETED 는 종착이라 중복해도 OK.
> `return` 으로 silent — 예외보다 명확.

> 💡 **왜 hard delete X?**
> Foreign Key 깨짐 (`orders`, `payments` 등이 user_id 참조). soft delete (`status='DELETED'` + email anonymize).
> 자세히: [[../database/users-table|database]] §soft delete.

---

## 7. `changeRole()` — 관리자 전용

```java
public void changeRole(Role newRole, Instant now, UserId changedBy) {
    if (status == UserStatus.DELETED)
        throw new IllegalStateException("deleted user");
    var oldRole = this.role;
    this.role = newRole;
    events.add(new UserRoleChanged(id, oldRole, newRole, changedBy, now));
}
```

> 💡 **왜 `changedBy` 인자?**
> Role 변경은 audit critical. **누가 변경했는지** event 에 포함 — listener 가 `role_change_history` 에 INSERT.

> 💡 **왜 권한 검사 (`changedBy.role == ADMIN`) 안 함?**
> 도메인 layer 는 "누가 호출 가능한가" 안 봄. Application Service (`@PreAuthorize`) 가 책임.
> 도메인은 "어떻게 변경되는가" 만.

---

## 8. Query Methods (불변 — 외부 노출)

```java
public boolean isActive() { return status == UserStatus.ACTIVE; }
public boolean canLogin() { return status == UserStatus.ACTIVE; }
public boolean emailVerified() { return emailVerifiedAt != null; }
public boolean phoneVerified() { return phoneVerifiedAt != null; }
public boolean isLocalAccount() { return providerType == SocialProviderType.LOCAL; }

public UserId id() { return id; }
public Email email() { return email; }
public Optional<PhoneNumber> phone() { return Optional.ofNullable(phone); }
public String name() { return name; }
public UserStatus status() { return status; }
public Role role() { return role; }
public SocialProviderType providerType() { return providerType; }
public Optional<String> externalId() { return Optional.ofNullable(externalId); }
public PasswordHash currentPasswordHash() { return passwordHash; }
public Instant createdAt() { return createdAt; }
```

> 💡 **왜 `Optional` 반환?**
> nullable 컬럼 (phone, externalId) 은 명시적 Optional — 호출자가 null 처리를 강제됨.
> 단, 반환에만 사용. 필드 / 인자에 Optional X — [[../../pitfalls/null-safety#5]] 참고.

---

## 9. `pullDomainEvents()` — 이벤트 회수

```java
public List<DomainEvent> pullDomainEvents() {
    var copy = List.copyOf(events);
    events.clear();
    return copy;
}
```

> 💡 **왜 immutable copy?**
> caller 가 변경 못 함. Aggregate 의 events 는 도메인이 책임.

> 💡 **왜 `clear()` 하는가?**
> "pull" = "회수". 한 번 회수한 이벤트는 다시 안 옴.
> 두 번 publish 방지 (idempotent X — caller 의 책임).

> 💡 **왜 `pull` 이라는 이름?**
> `getEvents()` 보다 의도 명확. **"가져가고 비움"** 의미.
> `flushEvents()`, `releaseEvents()` 도 OK — 본 vault 는 `pull`.

---

## 10. `reconstitute()` — Adapter 의 매핑 전용

```java
public static User reconstitute(UserId id, Email email, PhoneNumber phone,
                                PasswordHash hash, String name,
                                UserStatus status, Role role,
                                SocialProviderType providerType, String externalId,
                                Instant emailVerifiedAt, Instant phoneVerifiedAt,
                                Instant createdAt) {
    return new User(id, email, phone, hash, name, status, role,
                    providerType, externalId,
                    emailVerifiedAt, phoneVerifiedAt, createdAt);
}
```

> 💡 **왜 register 와 다른 메서드?**
> `register` 는 신규 가입 — `UserRegistered` event 발행, status=PENDING.
> `reconstitute` 는 **DB 에서 가져온 기존 user 를 도메인 객체로 매핑** — event 발행 X, status 그대로.

> 💡 **왜 public static?**
> Adapter (JPA / MyBatis) 가 호출해야 함. 단 도메인 외부 (Service, Controller) 가 호출 X — convention.
> 더 엄격하면 `package-private` 으로 (같은 패키지에 Adapter 있을 때) 또는 Builder 패턴.

---

## 11. 함정 모음

### 함정 1 — Setter public 노출
`user.setStatus(DELETED)` 누구나 가능 = 상태 머신 무의미. **반드시 메서드 통해**.

### 함정 2 — 도메인 안에서 외부 호출
```java
public void verifyEmail() {
    smsService.send(...);    // ❌ — 외부 의존
}
```
**도메인 이벤트만** 발행. listener 가 외부 호출.

### 함정 3 — events.clear() 안 함
`pullDomainEvents` 여러 번 호출 시 같은 이벤트 다중 publish. **clear 필수**.

### 함정 4 — `reconstitute` 가 event 발행
DB 에서 user 가져올 때마다 `UserRegistered` event = 무한 알림. **reconstitute 는 event X**.

### 함정 5 — Aggregate 가 너무 큼
`User` 에 `orders` / `posts` / `followers` 다 들고 있음 → 한 트랜잭션이 많은 entity 로드.
**ID 참조 + 별도 Aggregate** ([[aggregate-boundaries]]).

### 함정 6 — 도메인이 JPA Entity
`@Entity @Table(...)` 어노테이션 = ORM 종속. **도메인 ↔ Adapter 분리**.

### 함정 7 — `changeRole` 같은 critical 변경에 audit 없음
who / when / why 추적 불가. event 에 `changedBy` 포함 + listener 가 audit table INSERT.

### 함정 8 — `@Version` 의 위치
도메인은 모름. JPA Entity 가 `@Version` 가지고 — JPA layer 의 동시성 처리.

---

## 12. 관련

- [[domain-model|↑ domain-model hub]]
- [[value-objects]] — Email / PasswordHash / UserId / Phone
- [[domain-events]] — UserRegistered / UserEmailVerified / ...
- [[../enums/user-status]] — status 값 의미
- [[aggregate-boundaries]] — 왜 약관 동의가 User 안에 없는가
- [[../signup-impl]] — `User.register` 호출 흐름
