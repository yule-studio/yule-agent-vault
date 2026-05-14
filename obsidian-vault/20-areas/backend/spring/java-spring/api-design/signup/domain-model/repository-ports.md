---
title: "Repository Ports — Port-Adapter 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:17:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - repository
  - port-adapter
---

# Repository Ports — Port-Adapter 패턴

**[[domain-model|↑ domain-model hub]]**

> 도메인이 영속 (저장소 / 외부 시스템) 의존을 추상화. **interface 만** 도메인에, 구현은 infrastructure.

---

## 1. Port 의 책임

```
[Domain]
   │ 의존
   ▼
[Repository Port (interface)]   ← 도메인에 위치
   ▲ implements
   │
[JPA / MyBatis Adapter]         ← infrastructure 에 위치
   │ uses
   ▼
[PostgreSQL]
```

**도메인** 은 interface 만 보임. 구체적 SQL / JPA / ORM 알 필요 X.

---

## 2. auth 도메인의 5 Port

```java
// domain/user/UserRepository.java
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    Optional<User> findByPhone(PhoneNumber phone);
    Optional<User> findByProviderAndExternalId(SocialProviderType provider, String externalId);
    boolean existsByEmail(Email email);
    boolean existsByPhone(PhoneNumber phone);
    User save(User user);
    Optional<User> findById(UserId id);
}

// domain/auth/RefreshTokenRepository.java
public interface RefreshTokenRepository {
    RefreshToken save(RefreshToken token);
    Optional<RefreshToken> findById(RefreshTokenId id);
    Optional<RefreshToken> findByTokenHash(String hash);
    void revokeAllForUser(UserId userId);
}

// domain/auth/EmailVerificationTokenRepository.java
public interface EmailVerificationTokenRepository {
    EmailVerificationToken save(EmailVerificationToken t);
    Optional<EmailVerificationToken> findByTokenHash(String hash);
    void revokeAllActiveForUser(UserId userId);
    int countActiveForUserSince(UserId userId, Instant since);
}

// domain/auth/PhoneVerificationCodeRepository.java
public interface PhoneVerificationCodeRepository {
    PhoneVerificationCode save(PhoneVerificationCode code);
    Optional<PhoneVerificationCode> findActiveByPhone(PhoneNumber phone);
    void revokeAllActiveByPhone(PhoneNumber phone);
}

// domain/auth/PasswordResetTokenRepository.java
public interface PasswordResetTokenRepository {
    PasswordResetToken save(PasswordResetToken t);
    Optional<PasswordResetToken> findByTokenHash(String hash);
    void revokeAllActiveForUser(UserId userId);
    int countActiveForUserSince(UserId userId, Instant since);
}
```

→ 각 Aggregate 마다 별도 port. 책임 분리.

---

## 3. 왜 도메인 layer 에 interface

### 3.1 의존성 역전 (Dependency Inversion)

```
❌ 도메인이 JPA 의존
[Domain] ──→ [JpaRepository]
                  ↓
              [PostgreSQL]

✅ 의존성 역전
[Domain] ──→ [Port (interface)]
              ▲
              │ implements
[JpaAdapter] ─┘
   ↓
[PostgreSQL]
```

→ ORM 변경 시 (JPA → MyBatis → JOOQ) **도메인 코드 0 변경**. Adapter 만 교체.

### 3.2 Test 친화

```java
// 단위 테스트
class SignupUseCaseTest {
    UserRepository users = mock(UserRepository.class);    // 가벼운 mock
    SignupUseCase sut = new SignupUseCase(users, ...);

    @Test
    void test() {
        when(users.existsByEmail(...)).thenReturn(false);
        sut.handle(...);
    }
}
```

→ DB / Spring 컨텍스트 없이 unit test. ms 단위.

---

## 4. Port 의 메서드 명명 규칙

| 패턴 | 의미 | 예 |
| --- | --- | --- |
| `find*` | 조회 (`Optional`) | `findByEmail`, `findById` |
| `findAll*` | 다건 조회 | `findAllByStatus` |
| `exists*` | boolean | `existsByEmail` |
| `count*` | long | `countByStatus` |
| `save` | INSERT / UPDATE 통합 | `save(user)` |
| `delete*` | DELETE | `delete(user)`, `deleteById(id)` |

**금지**:
- `get*` — JPA 의 `getById` 는 lazy proxy 반환. 헷갈림. `find*` 만.
- `findByEmailIgnoreCase` — Spring Data Repository 의 명명 누수. **Port 는 도메인 언어** (`findByEmail` — Email VO 가 case-insensitive 보장).

---

## 5. Adapter 패턴

```java
// infrastructure/persistence/jpa/user/JpaUserRepositoryAdapter.java
@Repository
@RequiredArgsConstructor
public class JpaUserRepositoryAdapter implements UserRepository {

    private final UserJpaRepository spring;        // Spring Data Repository (별도)
    private final Clock clock;

    @Override
    public Optional<User> findByEmail(Email email) {
        return spring.findByEmailIgnoreCase(email.value())
            .map(this::toDomain);
    }

    @Override
    public User save(User user) {
        var now = Instant.now(clock);
        var entity = spring.findById(user.id().value())
            .map(e -> { e.apply(user, now); return e; })
            .orElseGet(() -> new UserJpaEntity(...));
        return toDomain(spring.save(entity));
    }

    private User toDomain(UserJpaEntity e) {
        return User.reconstitute(
            new UserId(e.getId()),
            new Email(e.getEmail()),
            ...
        );
    }
}
```

**책임 분리**:
- **Spring Data `UserJpaRepository`** — JPA 메서드 (`findByEmailIgnoreCase` 등). infra 내부.
- **`JpaUserRepositoryAdapter`** — 도메인 port 구현 + 매핑. 도메인이 보는 인터페이스.

> 💡 **왜 두 단계?**
> Spring Data 가 자동 생성하는 메서드는 JPA 의존. 도메인이 직접 사용하면 안 됨.
> Adapter 가 한 단계 추가해서 도메인 ↔ JPA 매핑 책임.

→ MyBatis 변형: [[../../database/mybatis#10.5.1]]

---

## 6. `save` 의 INSERT/UPDATE 분기

```java
@Override
public User save(User user) {
    var entity = spring.findById(user.id().value())
        .map(e -> { e.apply(user, now); return e; })          // UPDATE
        .orElseGet(() -> new UserJpaEntity(...));              // INSERT
    return toDomain(spring.save(entity));
}
```

> 💡 **왜 findById 후 분기?**
> JPA 의 `save` 가 자체로 detached entity 매핑은 미묘 — explicit 가 명확.

> 💡 **`entity.apply(user, now)`** — Entity 가 도메인 객체 보고 자신을 갱신. setter 호출 X (Entity 의 package-private 메서드).

---

## 7. 트랜잭션 — Adapter 가 책임 X

```java
@Override
public User save(User user) {
    // ❌ @Transactional 안 붙임
    return spring.save(toEntity(user));
}
```

→ 트랜잭션은 **Application Service 의 책임**. Adapter 는 그 안에서 실행되는 작업 단위만.

```java
// Application
@Transactional
public void signup(SignupCommand cmd) {
    users.save(user);              // Adapter — 트랜잭션 안에서 실행
    events.publish(...);
}
```

---

## 8. Adapter 의 예외 변환

```java
@Override
public User save(User user) {
    try {
        return toDomain(spring.save(toEntity(user)));
    } catch (DataIntegrityViolationException e) {
        // JPA 예외 → 도메인 예외 변환 (선택 — Service 가 잡아도 OK)
        throw new EmailAlreadyExistsException(user.email());
    }
}
```

옵션:
- **Adapter 에서 변환** — 도메인이 깨끗
- **Service 에서 잡고 변환** — Adapter 가 단순

본 vault: **Service 에서 잡음** (Adapter 는 단순 매핑만). 단 ORM-specific 예외는 Adapter 에서 도메인 친화 예외로 wrap 권장.

---

## 9. Port 분리 — `XRepository` vs `XQueryRepository`

```java
public interface UserRepository {                     // CUD + 단순 read
    Optional<User> findByEmail(Email email);
    User save(User user);
}

public interface UserQueryRepository {                // 복잡 검색 / 보고서
    Page<UserSummary> search(UserSearchCriteria c, Pageable p);
    long countByStatus(UserStatus status);
    List<MonthlyUserStats> monthlyStats(int year);
}
```

→ JPA only / MyBatis only / 공존 모드에서 유연.
- 공존: `UserRepository` = JPA Adapter, `UserQueryRepository` = MyBatis Adapter

자세히: [[../../api-design#0.5.4]] / [[../../database/jpa-mybatis-coexist#9.5]].

---

## 10. 함정 모음

### 함정 1 — 도메인 layer 가 Spring Data 직접 사용
`UserJpaRepository` 가 도메인 layer 에 있음. ORM 의존성 누수. **Port (interface) 만**.

### 함정 2 — `save` 의 INSERT/UPDATE 모호
Spring Data `save` 가 detached entity 매핑 어색. **explicit findById 후 분기**.

### 함정 3 — Adapter 에 `@Transactional`
트랜잭션 책임이 분산. **Service layer 만**.

### 함정 4 — Port 메서드가 너무 많음
`findByEmailAndStatusAndCreatedAfter` 같은 메서드 폭증. **Specification / Query Repository 분리**.

### 함정 5 — Port 의 인자가 ORM 타입
`findBySpec(Specification<UserJpaEntity>)` — Specification 자체가 JPA 의존. **도메인 친화 type** (`UserSearchCriteria` record).

### 함정 6 — Repository 가 도메인 외 객체 반환
`UserJpaEntity` 반환 → 도메인이 JPA 노출. **항상 `User` 도메인 객체** 반환.

### 함정 7 — getById vs findById
`getById` 는 JPA proxy 반환 (lazy). 통합 시점에 LazyInit 위험. **`findById` (eager) 만**.

### 함정 8 — equals / hashCode 없는 JPA Entity
`Set<UserJpaEntity>` 사용 시 같은 ID 도 다른 객체. ID 기반 equals 추가.

### 함정 9 — Port 가 페이지네이션 안 함
`findAllByXyz` 가 전체 반환 — DB 부하. **항상 `Pageable` / cursor**.

### 함정 10 — Port 의 메서드가 너무 generic
`update(User user)` — 어떤 필드 변경인지 불명. **명시적 도메인 메서드** (`User.changeName`) + port 는 단순 `save`.

---

## 11. 관련

- [[domain-model|↑ domain-model hub]]
- [[../../database/jpa#11. Repository Adapter 패턴]]
- [[../../database/mybatis#10. Repository Adapter]]
- [[../../database/jpa-mybatis-coexist#9.5 분담 패턴]]
- [[../../api-design#0.5 ORM 정책]]
