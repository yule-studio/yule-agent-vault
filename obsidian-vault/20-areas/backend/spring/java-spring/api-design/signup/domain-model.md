---
title: "signup §2 — 도메인 모델 + 규칙 + 상태 전이"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:36:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - domain
---

# signup §2 — 도메인 모델 + 규칙 + 상태 전이

**[[signup|↑ signup hub]]**  ·  ← [[requirements]]  ·  → [[architecture]]

---

## 1. 클래스 다이어그램

```
┌─────────────────────────────────┐
│ User (Aggregate Root)           │
├─────────────────────────────────┤
│ - id: UserId                    │
│ - email: Email                  │ ──┐ Value Object (record, 불변)
│ - passwordHash: PasswordHash    │ ──┤
│ - name: String                  │   │
│ - status: UserStatus            │   │
│ - createdAt: Instant            │   │
├─────────────────────────────────┤   │
│ + register(...)         (static)│   │
│ + changePassword(...)           │   │
│ + verifyEmail()                 │   │
│ + currentPasswordHash()         │   │
│ + pullDomainEvents()            │   │
└──────┬──────────────────────────┘   │
       │ raises                       │
       ▼                              │
┌─────────────────────────────────┐   │
│ DomainEvent (sealed interface)  │ <─┘
├─────────────────────────────────┤
│ - UserRegistered                │
│ - UserEmailVerified             │
│ - UserPasswordChanged           │
└─────────────────────────────────┘
```

---

## 2. Value Objects

### 2.1 Email — RFC 5322 (단순화)

```java
public record Email(String value) {
    private static final Pattern EMAIL = Pattern.compile(
        "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"
    );
    public Email {
        if (value == null || !EMAIL.matcher(value).matches())
            throw new IllegalArgumentException("invalid email: " + value);
        if (value.length() > 254) throw new IllegalArgumentException("email too long");
    }
}
```

**규칙**:
- 생성 시 검증 (compact constructor)
- 정규화 (소문자) 는 호출자 책임 — Email 은 받은 값 그대로 보존 (대문자도 valid)
- DB 의 `lower(email)` UNIQUE 가 case-insensitive 보장

### 2.2 PasswordHash — argon2 해시만

```java
public record PasswordHash(String value) {
    public PasswordHash {
        if (value == null || !value.startsWith("$argon2"))
            throw new IllegalArgumentException("not an argon2 hash");
    }
}
```

**규칙**:
- **평문 절대 X** — `$argon2id$...` 같은 PHC 형식만
- `Password` 라는 이름의 클래스 (평문 들고 다님) **금지**
- 평문은 method 파라미터 (`String rawPassword`) 로만 잠깐 존재했다 사라짐

### 2.3 UserId — ULID

```java
public record UserId(String value) {
    public UserId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("UserId must be ULID (26 chars)");
    }
}
```

**왜 ULID**:
- 시간순 정렬 (DB B-tree 친화)
- 추측 불가 (enumeration 방지)
- 26 char string (UUID 보다 짧음)
- DB 독립 (app-side 발급)

---

## 3. User Aggregate

```java
public final class User {

    private final UserId id;
    private final Email email;
    private PasswordHash passwordHash;          // changePassword 로만 변경
    private final String name;
    private UserStatus status;
    private final Instant createdAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private User(UserId id, Email email, PasswordHash hash, String name,
                 UserStatus status, Instant createdAt) {
        this.id = id; this.email = email; this.passwordHash = hash;
        this.name = name; this.status = status; this.createdAt = createdAt;
    }

    /** 신규 가입 — PENDING_VERIFICATION 상태로. */
    public static User register(UserId id, Email email, PasswordHash hash,
                                String name, Instant now) {
        if (name == null || name.isBlank() || name.length() > 100)
            throw new IllegalArgumentException("invalid name");
        var u = new User(id, email, hash, name, UserStatus.PENDING_VERIFICATION, now);
        u.events.add(new UserRegistered(id, email, now));
        return u;
    }

    /** 이메일 인증 완료 — PENDING_VERIFICATION 일 때만. */
    public void verifyEmail() {
        if (status != UserStatus.PENDING_VERIFICATION)
            throw new IllegalStateException("not pending: " + status);
        status = UserStatus.ACTIVE;
        events.add(new UserEmailVerified(id, email, Instant.now()));
    }

    /** 패스워드 변경 — DELETED 외 모두 가능. */
    public void changePassword(PasswordHash newHash) {
        if (status == UserStatus.DELETED)
            throw new IllegalStateException("deleted user");
        this.passwordHash = newHash;
        events.add(new UserPasswordChanged(id, Instant.now()));
    }

    /** 일시 정지 — ACTIVE 만 가능. */
    public void suspend() {
        if (status != UserStatus.ACTIVE)
            throw new IllegalStateException("only ACTIVE can be suspended");
        status = UserStatus.SUSPENDED;
    }

    /** 정지 해제 — SUSPENDED 만 가능. */
    public void unsuspend() {
        if (status != UserStatus.SUSPENDED)
            throw new IllegalStateException("not suspended");
        status = UserStatus.ACTIVE;
    }

    /** soft delete — 모든 상태에서 가능. */
    public void delete() {
        if (status == UserStatus.DELETED) return;       // 멱등
        status = UserStatus.DELETED;
    }

    // queries
    public boolean isActive() { return status == UserStatus.ACTIVE; }
    public boolean canLogin() { return status == UserStatus.ACTIVE; }

    public UserId id() { return id; }
    public Email email() { return email; }
    public String name() { return name; }
    public UserStatus status() { return status; }
    public Instant createdAt() { return createdAt; }
    public PasswordHash currentPasswordHash() { return passwordHash; }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events); events.clear(); return copy;
    }

    /** JPA / MyBatis Adapter 의 reconstitution 용. 도메인 동작 X. */
    public static User reconstitute(UserId id, Email email, PasswordHash hash,
                                    String name, UserStatus status, Instant createdAt) {
        return new User(id, email, hash, name, status, createdAt);
    }
}
```

---

## 4. UserStatus — 상태 머신

```java
public enum UserStatus {
    PENDING_VERIFICATION,    // 가입 직후, 이메일 미인증
    ACTIVE,                  // 인증 완료, 로그인 가능
    SUSPENDED,               // 관리자 정지
    DELETED                  // soft delete
}
```

### 4.1 상태 전이 다이어그램

```
                              ┌─────────────┐
                              │   (없음)    │
                              └──────┬──────┘
                                     │ register()
                                     ▼
                  ┌───────────────────────────────┐
                  │   PENDING_VERIFICATION         │
                  │                                │
                  │ - 로그인 X                      │
                  │ - 이메일 인증 메일 발송 대상       │
                  └──────┬─────────────────────────┘
                         │ verifyEmail()
                         ▼
        ┌────────────────────────────────────────┐
        │   ACTIVE                                │
        │                                         │
        │ - 정상 사용자                            │
        │ - 로그인 OK                              │
        │ - changePassword / changeName OK         │
        └────────┬───────────────────────────────┘
                 │                          │
                 │ suspend()                │ delete()
                 ▼                          ▼
        ┌────────────────┐         ┌────────────────┐
        │   SUSPENDED    │ ─delete─▶   DELETED      │
        │                │         │                │
        │ - 로그인 X      │         │ - 모든 작업 X   │
        │ - 관리자만 해제  │         │ - 복구 X       │
        └────────┬───────┘         └────────────────┘
                 │
                 │ unsuspend()
                 ▼
              (ACTIVE)
```

### 4.2 상태 전이 매트릭스

| 현재 \ 다음 | PENDING_VERIFICATION | ACTIVE | SUSPENDED | DELETED |
| --- | --- | --- | --- | --- |
| **PENDING_VERIFICATION** | — | `verifyEmail()` ✅ | ❌ | `delete()` ✅ |
| **ACTIVE** | ❌ | — | `suspend()` ✅ | `delete()` ✅ |
| **SUSPENDED** | ❌ | `unsuspend()` ✅ | — | `delete()` ✅ |
| **DELETED** | ❌ | ❌ | ❌ | 멱등 (no-op) |

---

## 5. 도메인 규칙 (불변식)

각 규칙은 도메인 객체 안에서 자체 검증. Service / Repository 가 검증 책임 가지지 않음.

1. **email 은 RFC 5322 형식 + 254 자 이내** — `Email` value object 가 보장
2. **password hash 는 argon2 PHC 형식** — `PasswordHash` value object 가 보장
3. **id 는 26자 ULID** — `UserId` value object 가 보장
4. **name 은 1~100자, blank 금지** — `User.register` 검증
5. **신규 가입은 항상 PENDING_VERIFICATION** — `register()` 만 user 생성, 직접 ACTIVE 못 만듦
6. **PENDING → ACTIVE 만 가능, 다른 경로 X** — `verifyEmail()` 만
7. **DELETED 는 종착역** — 어디서든 갈 수 있지만, 거기서 안 나옴
8. **DELETED 의 패스워드 변경 X** — `changePassword()` 가 거절
9. **모든 상태 변경은 도메인 이벤트 발행** — listener 가 후속 처리

---

## 6. Domain Events

```java
public sealed interface DomainEvent permits
    UserRegistered, UserEmailVerified, UserPasswordChanged {
    Instant occurredAt();
}

public record UserRegistered(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserEmailVerified(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserPasswordChanged(UserId userId, Instant occurredAt) implements DomainEvent {}
```

`pullDomainEvents()` 으로 trigger 한 측 (UseCase) 이 받아서 `ApplicationEventPublisher.publishEvent(...)` 로 발행.

---

## 7. Repository Port — 도메인 layer 의 interface

```java
// src/main/java/com/example/shop/domain/user/UserRepository.java
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    boolean existsByEmail(Email email);
    User save(User user);
    Optional<User> findById(UserId id);
}

/** 검색 / 통계 등 — JPA + MyBatis 공존 모드에서 분리. */
public interface UserQueryRepository {
    Page<UserSummary> search(UserSearchCriteria c, Pageable p);
    long countByStatus(UserStatus status);
}
```

→ 본 레시피의 CUD 는 `UserRepository` 만 사용. 검색 / 통계 화면이 생기면 `UserQueryRepository` 추가.

---

## 8. 관련

- [[signup|↑ signup hub]]
- [[requirements]] — 이전 (§1)
- [[architecture]] — 다음 (§3)
- [[../email-verification]] — `verifyEmail()` 호출
- [[../password-reset]] — `changePassword()` 호출
