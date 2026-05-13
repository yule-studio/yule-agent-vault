---
title: "회원가입 (Signup) — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - signup
---

# 회원가입 (Signup) — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Java 21 + Spring Boot 3 / argon2id / outbox / 전체 코드 |

**[[api-design|↑ api-design hub]]**

---

## 1. 무엇을 만드는가

`POST /api/v1/auth/signup` — 이메일 + 패스워드 + 이름으로 신규 계정 생성.

### 1.1 요청 / 응답

```http
POST /api/v1/auth/signup
Content-Type: application/json

{
  "email": "alice@example.com",
  "password": "Tr0ub4dor&3-with-len",
  "name": "Alice",
  "termsAgreed": true,
  "marketingAgreed": false
}
```

```http
HTTP/1.1 201 Created
{
  "data": {
    "userId": "01HZ9X...",
    "email": "alice@example.com",
    "status": "PENDING_VERIFICATION",
    "createdAt": "2026-05-14T09:15:23Z"
  }
}
```

부수효과: 이메일 인증 메일 발송 (상세 — `email-verification` 레시피).

### 1.2 비기능

- **idempotent 아님** — 같은 이메일 재시도는 409 (또는 enumeration 차단 정책에 따라 200)
- **p95 < 500ms** (argon2id 가 100~300ms 차지)
- **이메일 unique** — DB 제약 + 비즈니스 검증 이중
- **평문 패스워드 로그/저장 절대 X** — 입력 즉시 해시
- **이메일 발송은 트랜잭션 밖** — outbox

---

## 2. 도메인 모델 (OOP)

### 2.1 클래스 다이어그램

```
┌─────────────────────────────────┐
│ User (Aggregate Root)           │
│ - id: UserId                    │
│ - email: Email                  │ ──┐ Value Object (record, 불변)
│ - passwordHash: PasswordHash    │ ──┤
│ - name: String                  │   │
│ - status: UserStatus            │   │
│ - createdAt: Instant            │   │
│ + changePassword(...)           │   │
│ + verifyEmail()                 │   │
└─────────────────────────────────┘   │
       △                              │
       │ raises                       │
       ▼                              │
┌─────────────────────────────────┐   │
│ UserRegistered (DomainEvent)    │ <─┘
└─────────────────────────────────┘
```

### 2.2 Value Objects (record)

```java
// src/main/java/com/example/shop/domain/user/Email.java
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

// src/main/java/com/example/shop/domain/user/PasswordHash.java
// 해시된 패스워드 자체가 도메인 객체 — 평문은 들고 다니지 않는다.
public record PasswordHash(String value) {
    public PasswordHash {
        if (value == null || !value.startsWith("$argon2"))
            throw new IllegalArgumentException("not an argon2 hash");
    }
}

// src/main/java/com/example/shop/domain/user/UserId.java
public record UserId(String value) {
    public UserId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("UserId must be ULID (26 chars)");
    }
}
```

> **함정**: `Password` 라는 이름의 클래스가 평문을 들고 다니게 만들지 말 것. 평문은 메서드 파라미터 (String) 로만 잠깐 존재했다 사라진다.

### 2.3 User Aggregate

```java
// src/main/java/com/example/shop/domain/user/User.java
public final class User {
    private final UserId id;
    private final Email email;
    private PasswordHash passwordHash;
    private final String name;
    private UserStatus status;
    private final Instant createdAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private User(UserId id, Email email, PasswordHash hash, String name,
                 UserStatus status, Instant createdAt) {
        this.id = id; this.email = email; this.passwordHash = hash;
        this.name = name; this.status = status; this.createdAt = createdAt;
    }

    public static User register(UserId id, Email email, PasswordHash hash,
                                String name, Instant now) {
        if (name == null || name.isBlank() || name.length() > 100)
            throw new IllegalArgumentException("invalid name");
        var u = new User(id, email, hash, name, UserStatus.PENDING_VERIFICATION, now);
        u.events.add(new UserRegistered(id, email, now));
        return u;
    }

    public void changePassword(PasswordHash newHash) {
        this.passwordHash = newHash;
        events.add(new UserPasswordChanged(id, Instant.now()));
    }

    public void verifyEmail() {
        if (status != UserStatus.PENDING_VERIFICATION)
            throw new IllegalStateException("not pending: " + status);
        status = UserStatus.ACTIVE;
        events.add(new UserEmailVerified(id, email, Instant.now()));
    }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events);
        events.clear();
        return copy;
    }

    // getters
    public UserId id() { return id; }
    public Email email() { return email; }
    public PasswordHash currentPasswordHash() { return passwordHash; }
    public String name() { return name; }
    public UserStatus status() { return status; }
    public Instant createdAt() { return createdAt; }
    public boolean isActive() { return status == UserStatus.ACTIVE; }

    // JPA 매핑용 reconstitution (infrastructure 에서만 사용)
    public static User reconstitute(UserId id, Email email, PasswordHash hash,
                                    String name, UserStatus status, Instant createdAt) {
        return new User(id, email, hash, name, status, createdAt);
    }
}

public enum UserStatus { PENDING_VERIFICATION, ACTIVE, SUSPENDED, DELETED }
```

### 2.4 Domain Events

```java
// src/main/java/com/example/shop/domain/common/DomainEvent.java
public sealed interface DomainEvent permits UserRegistered, UserEmailVerified, UserPasswordChanged {
    Instant occurredAt();
}

// src/main/java/com/example/shop/domain/user/events/UserRegistered.java
public record UserRegistered(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserEmailVerified(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserPasswordChanged(UserId userId, Instant occurredAt) implements DomainEvent {}
```

### 2.5 Repository port

```java
// src/main/java/com/example/shop/domain/user/UserRepository.java
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    boolean existsByEmail(Email email);
    User save(User user);
    Optional<User> findById(UserId id);
}
```

---

## 3. 아키텍처 / 의존성 흐름

```
HTTP                   Application                Domain                 Infra
─────                  ───────────                ──────                 ─────
SignupController ───→  SignupUseCase ─────→  User.register()
                            │                      │
                            │                      │ raises UserRegistered
                            │                  ┌───┴────────┐
                            │                  ▼            │
                    UserRepository ◀── interface impl ── JpaUserRepository ──→ PostgreSQL
                            │
                            ▼
                    PasswordEncoder ◀── port impl ── Argon2PasswordEncoder
                            │
                            ▼
                    ApplicationEventPublisher ── @TransactionalEventListener(AFTER_COMMIT)
                                                       │
                                                       ▼
                                                 EmailOutbox
```

- Controller — DTO ↔ UseCase 변환만
- UseCase — `@Transactional` 경계 + 도메인 호출 + 저장
- 이메일 발송 — **트랜잭션 커밋 후** outbox 적재 (메일 발송 워커는 별도)

---

## 4. DB 선택 / 스키마

### 4.1 왜 PostgreSQL

| 후보 | 적합 | 이유 |
| --- | --- | --- |
| **PostgreSQL / MySQL** | ✅ | `UNIQUE (lower(email))` 한 줄로 race 차단. ACID. |
| MongoDB | ⚠️ | unique index 가능. 동시성 처리 미묘. |
| DynamoDB | ⚠️ | conditional put. query 패턴 PK 의존적. |
| Redis | ❌ | 영속 / 백업 / 관계 핵심엔 부적합 |

결론: **PostgreSQL**. email unique 동시성 보장이 DB constraint 로 사실상 공짜. 추후 order/payment 와 JOIN.

### 4.2 스키마

```sql
-- src/main/resources/db/migration/V1__create_users.sql
CREATE TABLE users (
  id              CHAR(26) PRIMARY KEY,
  email           VARCHAR(254) NOT NULL,
  password_hash   VARCHAR(255) NOT NULL,
  name            VARCHAR(100) NOT NULL,
  status          VARCHAR(30)  NOT NULL,
  created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- 핵심: case-insensitive unique
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

CREATE INDEX ix_users_status ON users (status) WHERE status <> 'DELETED';
```

> **함정**: `email VARCHAR(254) UNIQUE` 만 걸면 `Alice@x.com`/`alice@x.com` 둘 다 통과. `lower(email)` 표현식 인덱스 필수.

### 4.3 왜 ULID

| 후보 | 장점 | 단점 |
| --- | --- | --- |
| `BIGSERIAL` | 단순 | enumeration 위험 |
| UUIDv4 | 추측 불가 | 정렬 안 됨 / 쓰기 비용 |
| **ULID / UUIDv7** | 시간순 + 추측 불가 | 26 char string |

권장: ULID 또는 UUIDv7.

---

## 5. 보안 / 암호화

### 5.1 패스워드 해시 — argon2id 권장

| 알고리즘 | 권장 | 비고 |
| --- | --- | --- |
| **argon2id** | ✅ OWASP 2024 1순위 | m=64MB t=3 p=4 |
| **bcrypt** | ✅ 대안 | cost ≥ 12. 72-byte 입력 제한 |
| scrypt | △ | argon2 보다 우선 낮음 |
| PBKDF2-HMAC-SHA256 | △ | iterations ≥ 600,000 |
| MD5 / SHA-1 / SHA-256 | ❌❌❌ | 패스워드 해시 X. 초당 수십억 시도 가능 |

### 5.2 password4j 로 argon2id

```java
// src/main/java/com/example/shop/domain/user/PasswordEncoder.java (port)
public interface PasswordEncoder {
    String encode(String plain);
    boolean matches(String plain, String hash);
}
```

```java
// src/main/java/com/example/shop/infrastructure/security/Argon2PasswordEncoder.java
@Component
public class Argon2PasswordEncoder implements PasswordEncoder {

    // OWASP 2024: m=64MB, t=3, p=4
    private static final int MEM_KB = 65536;
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

    /** 운영 중 파라미터 강화 시 — 로그인 성공 후 자동 rehash 판정에 사용 */
    public boolean needsRehash(String hash) {
        var current = Argon2Function.getInstanceFromHash(hash);
        return current.getMemory() != MEM_KB
            || current.getIterations() != ITER
            || current.getParallelism() != PAR;
    }
}
```

> **함정 1**: Spring Security 의 `BCryptPasswordEncoder` 도 OK. argon2id 가 새 프로젝트 1순위.
> **함정 2**: bcrypt 는 72-byte 까지만 의미 있음 → 길면 잘림. argon2id 권장.

### 5.3 입력 검증 / 정책

NIST 800-63B (2017+) 의 패스워드 정책:
- **최소 8자** (길수록 좋음)
- **최대 64+자** 허용 (자르지 말 것)
- **복잡도 강제 X** — 길이 우선
- **알려진 유출 패스워드 차단** (haveibeenpwned k-anonymity)

```java
// src/main/java/com/example/shop/application/user/PasswordPolicy.java
@Component
public class PasswordPolicy {

    private final PwnedPasswordChecker pwnedCheck;     // optional bean
    public PasswordPolicy(PwnedPasswordChecker pwnedCheck) { this.pwnedCheck = pwnedCheck; }

    public void validate(String plain) {
        if (plain == null || plain.length() < 8 || plain.length() > 128)
            throw new WeakPasswordException("password length must be 8-128");
        if (plain.indexOf(' ') >= 0)
            throw new WeakPasswordException("null byte in password");
        if (pwnedCheck.isPwned(plain))
            throw new WeakPasswordException("password found in breach");
    }
}
```

---

## 6. 구현 — Spring Boot

### 6.1 패키지 구조

```
com.example.shop
├── domain/user/
│   ├── User.java
│   ├── Email.java
│   ├── PasswordHash.java
│   ├── UserId.java
│   ├── UserStatus.java
│   ├── UserRepository.java
│   ├── PasswordEncoder.java
│   ├── events/UserRegistered.java
│   └── exceptions/EmailAlreadyExistsException.java
├── application/user/
│   ├── SignupCommand.java
│   ├── SignupUseCase.java
│   └── PasswordPolicy.java
├── infrastructure/
│   ├── persistence/jpa/user/
│   │   ├── UserJpaEntity.java
│   │   ├── UserJpaRepository.java
│   │   └── JpaUserRepositoryAdapter.java
│   ├── security/Argon2PasswordEncoder.java
│   └── messaging/UserRegisteredEmailListener.java
└── presentation/api/v1/auth/
    ├── SignupController.java
    ├── SignupRequest.java
    ├── SignupResponse.java
    └── ApiExceptionHandler.java
```

### 6.2 DTO (record)

```java
// src/main/java/com/example/shop/presentation/api/v1/auth/SignupRequest.java
public record SignupRequest(
    @Email @NotBlank @Size(max = 254) String email,
    @NotBlank @Size(min = 8, max = 128) String password,
    @NotBlank @Size(max = 100) String name,
    @AssertTrue(message = "must agree to terms") boolean termsAgreed,
    boolean marketingAgreed
) {
    // toString 마스킹 — 로그에 평문 password 노출 차단
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s, termsAgreed=%s]"
            .formatted(email, name, termsAgreed);
    }
}

// src/main/java/com/example/shop/presentation/api/v1/auth/SignupResponse.java
public record SignupResponse(String userId, String email, String status, Instant createdAt) {}
```

### 6.3 Command

```java
// src/main/java/com/example/shop/application/user/SignupCommand.java
public record SignupCommand(
    String email,
    String rawPassword,
    String name,
    boolean termsAgreed,
    boolean marketingAgreed
) {}
```

### 6.4 UseCase

```java
// src/main/java/com/example/shop/application/user/SignupUseCase.java
@Service
public class SignupUseCase {

    private final UserRepository users;
    private final PasswordEncoder passwordEncoder;
    private final PasswordPolicy passwordPolicy;
    private final IdGenerator idGenerator;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    public SignupUseCase(UserRepository users, PasswordEncoder passwordEncoder,
                         PasswordPolicy passwordPolicy, IdGenerator idGenerator,
                         Clock clock, ApplicationEventPublisher events) {
        this.users = users; this.passwordEncoder = passwordEncoder;
        this.passwordPolicy = passwordPolicy; this.idGenerator = idGenerator;
        this.clock = clock; this.events = events;
    }

    @Transactional
    public User handle(SignupCommand cmd) {
        if (!cmd.termsAgreed()) throw new IllegalArgumentException("terms must be agreed");
        passwordPolicy.validate(cmd.rawPassword());

        var email = new Email(cmd.email().trim().toLowerCase(Locale.ROOT));

        // 1차 비즈니스 검증. DB UNIQUE 가 최종 진실의 원천.
        if (users.existsByEmail(email)) throw new EmailAlreadyExistsException(email);

        var hash = new PasswordHash(passwordEncoder.encode(cmd.rawPassword()));
        var user = User.register(
            new UserId(idGenerator.next()),
            email,
            hash,
            cmd.name().trim(),
            Instant.now(clock)
        );

        User saved;
        try {
            saved = users.save(user);
        } catch (DataIntegrityViolationException e) {
            // race: 1차 검증 통과 후 다른 트랜잭션이 먼저 INSERT
            throw new EmailAlreadyExistsException(email);
        }

        saved.pullDomainEvents().forEach(events::publishEvent);
        return saved;
    }
}
```

### 6.5 JPA 영속

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/user/UserJpaEntity.java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(nullable = false, length = 254) private String email;
    @Column(name = "password_hash", nullable = false, length = 255) private String passwordHash;
    @Column(nullable = false, length = 100) private String name;
    @Column(nullable = false, length = 30) private String status;
    @Column(name = "created_at", nullable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;

    protected UserJpaEntity() {}    // JPA
    public UserJpaEntity(String id, String email, String passwordHash, String name,
                         String status, Instant createdAt, Instant updatedAt) {
        this.id = id; this.email = email; this.passwordHash = passwordHash;
        this.name = name; this.status = status;
        this.createdAt = createdAt; this.updatedAt = updatedAt;
    }
    // getters / package-private setters 생략 (Lombok 권장 시 @Getter @Setter)

    void apply(User u, Instant now) {
        this.passwordHash = u.currentPasswordHash().value();
        this.status = u.status().name();
        this.updatedAt = now;
    }
}

// src/main/java/com/example/shop/infrastructure/persistence/jpa/user/UserJpaRepository.java
public interface UserJpaRepository extends JpaRepository<UserJpaEntity, String> {
    @Query("select case when count(u) > 0 then true else false end " +
           "from UserJpaEntity u where lower(u.email) = lower(:email)")
    boolean existsByEmailIgnoreCase(@Param("email") String email);

    @Query("from UserJpaEntity u where lower(u.email) = lower(:email)")
    Optional<UserJpaEntity> findByEmailIgnoreCase(@Param("email") String email);
}

// src/main/java/com/example/shop/infrastructure/persistence/jpa/user/JpaUserRepositoryAdapter.java
@Repository
public class JpaUserRepositoryAdapter implements UserRepository {
    private final UserJpaRepository spring;
    private final Clock clock;

    public JpaUserRepositoryAdapter(UserJpaRepository spring, Clock clock) {
        this.spring = spring; this.clock = clock;
    }

    @Override public boolean existsByEmail(Email email) {
        return spring.existsByEmailIgnoreCase(email.value());
    }
    @Override public Optional<User> findByEmail(Email email) {
        return spring.findByEmailIgnoreCase(email.value()).map(this::toDomain);
    }
    @Override public Optional<User> findById(UserId id) {
        return spring.findById(id.value()).map(this::toDomain);
    }
    @Override public User save(User user) {
        var now = Instant.now(clock);
        var entity = spring.findById(user.id().value())
            .map(e -> { e.apply(user, now); return e; })
            .orElseGet(() -> new UserJpaEntity(
                user.id().value(), user.email().value(),
                user.currentPasswordHash().value(), user.name(),
                user.status().name(), user.createdAt(), now));
        return toDomain(spring.save(entity));
    }

    private User toDomain(UserJpaEntity e) {
        return User.reconstitute(
            new UserId(e.getId()), new Email(e.getEmail()),
            new PasswordHash(e.getPasswordHash()), e.getName(),
            UserStatus.valueOf(e.getStatus()), e.getCreatedAt()
        );
    }
}
```

> **함정**: 도메인 객체를 `@Entity` 로 만들면 편하지만 도메인이 JPA 에 종속. 본 레시피는 분리. 작은 프로젝트는 합쳐도 OK.

### 6.6 Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/auth/SignupController.java
@RestController
@RequestMapping("/api/v1/auth")
public class SignupController {
    private final SignupUseCase signup;
    public SignupController(SignupUseCase signup) { this.signup = signup; }

    @PostMapping("/signup")
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<SignupResponse> signup(@Valid @RequestBody SignupRequest req) {
        var user = signup.handle(toCommand(req));
        return ApiResponse.ok(new SignupResponse(
            user.id().value(),
            user.email().value(),
            user.status().name(),
            user.createdAt()
        ));
    }

    private static SignupCommand toCommand(SignupRequest r) {
        return new SignupCommand(r.email(), r.password(), r.name(),
                                 r.termsAgreed(), r.marketingAgreed());
    }
}
```

### 6.7 글로벌 예외 처리

```java
// src/main/java/com/example/shop/presentation/api/v1/ApiExceptionHandler.java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(EmailAlreadyExistsException.class)
    public ResponseEntity<ApiResponse<Void>> emailDup(EmailAlreadyExistsException e) {
        return ResponseEntity.status(409).body(ApiResponse.fail(
            new ApiError("EMAIL_ALREADY_EXISTS", e.getMessage())));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> validation(MethodArgumentNotValidException e) {
        Map<String, Object> details = e.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField,
                fe -> fe.getDefaultMessage() == null ? "invalid" : fe.getDefaultMessage(),
                (a, b) -> a));
        return ResponseEntity.status(422).body(ApiResponse.fail(
            new ApiError("VALIDATION_FAILED", "invalid request", details)));
    }

    @ExceptionHandler(WeakPasswordException.class)
    public ResponseEntity<ApiResponse<Void>> weakPw(WeakPasswordException e) {
        return ResponseEntity.status(422).body(ApiResponse.fail(
            new ApiError("WEAK_PASSWORD", e.getMessage())));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ApiResponse<Void>> bad(IllegalArgumentException e) {
        return ResponseEntity.status(400).body(ApiResponse.fail(
            new ApiError("INVALID_REQUEST", e.getMessage())));
    }
}
```

### 6.8 도메인 이벤트 → 이메일 outbox

```java
// src/main/java/com/example/shop/infrastructure/messaging/UserRegisteredEmailListener.java
@Component
public class UserRegisteredEmailListener {

    private final EmailOutboxRepository outbox;
    private final Clock clock;
    private final IdGenerator ids;

    public UserRegisteredEmailListener(EmailOutboxRepository outbox, Clock clock, IdGenerator ids) {
        this.outbox = outbox; this.clock = clock; this.ids = ids;
    }

    // 트랜잭션 커밋 후에만 outbox 에 적재
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(UserRegistered event) {
        outbox.enqueue(new EmailOutboxRow(
            ids.next(),
            event.email().value(),
            "verification",
            Map.of("userId", event.userId().value()),
            Instant.now(clock)
        ));
    }
}
```

> **핵심**: 이메일 발송은 **별도 워커 (Scheduler / Kafka consumer)** 가 outbox 에서 픽업. 트랜잭션 안에서 SMTP 호출 절대 X.

### 6.9 application.yml

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}          # TODO(secret): vault 주입
  jpa:
    hibernate:
      ddl-auto: validate              # 운영 절대 update / create-drop 금지
    properties:
      hibernate:
        jdbc.batch_size: 30
        order_inserts: true
  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    org.springframework.security: INFO
    com.example.shop: INFO

management:
  endpoint:
    configprops:
      show-values: never              # 패스워드 등 마스킹
```

### 6.10 IdGenerator / Clock 빈

```java
// src/main/java/com/example/shop/config/CommonConfig.java
@Configuration
public class CommonConfig {

    @Bean public Clock clock() { return Clock.systemUTC(); }

    @Bean public IdGenerator idGenerator() {
        return () -> ULID.random();   // io.azam.ulidj
    }
}

// src/main/java/com/example/shop/domain/common/IdGenerator.java
public interface IdGenerator { String next(); }
```

---

## 7. 트랜잭션 / 예외 / 검증 정리

### 7.1 트랜잭션 경계

```
@Transactional 시작 ─── SignupUseCase.handle
                        ├─ users.existsByEmail        (READ)
                        ├─ user = User.register(...)
                        ├─ users.save(user)
                        └─ events.publishEvent(...)    ← 큐잉만
@Transactional 커밋 ── DB COMMIT
                        └─ @TransactionalEventListener(AFTER_COMMIT)
                            └─ outbox.enqueue(...)     ← 새 트랜잭션
워커 (별도 프로세스) ──── outbox row 픽업 → SMTP 발송
```

### 7.2 예외 매핑

| 예외 | 상태 | 응답 |
| --- | --- | --- |
| `MethodArgumentNotValidException` | 422 | `VALIDATION_FAILED` + field 별 details |
| `EmailAlreadyExistsException` | 409 | `EMAIL_ALREADY_EXISTS` |
| `WeakPasswordException` | 422 | `WEAK_PASSWORD` |
| `IllegalArgumentException` (도메인) | 400 | `INVALID_REQUEST` |
| 그 외 | 500 | `INTERNAL_ERROR` (스택은 로그만) |

---

## 8. 테스트

### 8.1 단위 — 도메인

```java
// src/test/java/com/example/shop/domain/user/UserTest.java
class UserTest {
    @Test
    void register_raises_UserRegistered_event() {
        var u = User.register(
            new UserId("01HZ" + "X".repeat(22)),
            new Email("a@b.com"),
            new PasswordHash("$argon2id$v=19$m=65536,t=3,p=4$x$y"),
            "alice",
            Instant.parse("2026-05-14T00:00:00Z")
        );
        assertThat(u.status()).isEqualTo(UserStatus.PENDING_VERIFICATION);
        var events = u.pullDomainEvents();
        assertThat(events).hasSize(1).first().isInstanceOf(UserRegistered.class);
    }

    @Test
    void verifyEmail_moves_to_ACTIVE_only_when_PENDING() {
        var u = newPendingUser();
        u.verifyEmail();
        assertThat(u.status()).isEqualTo(UserStatus.ACTIVE);
        assertThatThrownBy(u::verifyEmail).isInstanceOf(IllegalStateException.class);
    }
}
```

### 8.2 단위 — UseCase (Mockito)

```java
// src/test/java/com/example/shop/application/user/SignupUseCaseTest.java
@ExtendWith(MockitoExtension.class)
class SignupUseCaseTest {

    @Mock UserRepository users;
    @Mock PasswordEncoder encoder;
    @Mock PasswordPolicy policy;
    @Mock IdGenerator ids;
    @Mock ApplicationEventPublisher events;

    Clock clock = Clock.fixed(Instant.parse("2026-05-14T00:00:00Z"), ZoneOffset.UTC);
    SignupUseCase sut;

    @BeforeEach
    void setup() {
        sut = new SignupUseCase(users, encoder, policy, ids, clock, events);
    }

    @Test
    void signs_up_new_user() {
        when(ids.next()).thenReturn("01HZ" + "X".repeat(22));
        when(users.existsByEmail(new Email("a@b.com"))).thenReturn(false);
        when(encoder.encode("p@ssword12")).thenReturn("$argon2id$x");
        when(users.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var user = sut.handle(new SignupCommand("a@b.com", "p@ssword12", "alice", true, false));

        assertThat(user.email().value()).isEqualTo("a@b.com");
        verify(events).publishEvent(any(UserRegistered.class));
    }

    @Test
    void rejects_duplicate_email() {
        when(users.existsByEmail(new Email("a@b.com"))).thenReturn(true);
        assertThatThrownBy(() ->
            sut.handle(new SignupCommand("a@b.com", "p@ssword12", "alice", true, false))
        ).isInstanceOf(EmailAlreadyExistsException.class);
    }
}
```

### 8.3 통합 — Testcontainers

```java
// src/test/java/com/example/shop/integration/SignupApiIT.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class SignupApiIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("shop_it").withUsername("test").withPassword("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry reg) {
        reg.add("spring.datasource.url", postgres::getJdbcUrl);
        reg.add("spring.datasource.username", postgres::getUsername);
        reg.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired TestRestTemplate rest;

    @Test
    void POST_signup_creates_user_and_returns_201() {
        var body = Map.of(
            "email", "alice@x.com",
            "password", "Tr0ub4dor!",
            "name", "Alice",
            "termsAgreed", true
        );
        var res = rest.postForEntity("/api/v1/auth/signup", body, Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(201);
        var data = (Map<?, ?>) res.getBody().get("data");
        assertThat(data.get("status")).isEqualTo("PENDING_VERIFICATION");
    }

    @Test
    void duplicate_email_returns_409() {
        var body = Map.of("email", "dup@x.com", "password", "Tr0ub4dor!",
                          "name", "x", "termsAgreed", true);
        rest.postForEntity("/api/v1/auth/signup", body, Map.class);
        var res2 = rest.postForEntity("/api/v1/auth/signup", body, Map.class);
        assertThat(res2.getStatusCode().value()).isEqualTo(409);
    }

    @Test
    void case_insensitive_duplicate_also_409() {
        rest.postForEntity("/api/v1/auth/signup",
            Map.of("email", "Alice@X.com", "password", "p@ssword12",
                   "name", "A", "termsAgreed", true), Map.class);
        var res = rest.postForEntity("/api/v1/auth/signup",
            Map.of("email", "alice@x.com", "password", "p@ssword12",
                   "name", "B", "termsAgreed", true), Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(409);
    }
}
```

---

## 9. 운영 체크리스트

- [ ] `password_hash` 가 `application.yml` / 로그에 노출 X (Hibernate SQL 로그 INFO 이상)
- [ ] argon2 파라미터가 서버 CPU/메모리에 맞는지 (`p95 signup latency` 모니터)
- [ ] `users.email` 의 `lower(email)` unique index 존재
- [ ] signup endpoint 에 **rate limiting** (봇 / enumeration 차단)
- [ ] 이메일 outbox 워커 라이브니스 + 알람 (대기 큐 길이)
- [ ] 트랜잭션 ID + user ID 가 access log 에 (`MDC`)
- [ ] hard delete X — soft delete (`status = DELETED`)
- [ ] 개인정보 컬럼 RDS 암호화 + 백업 암호화
- [ ] HTTPS 강제 + HSTS
- [ ] WAF / CAPTCHA (대량 가입 봇 대비)
- [ ] 침해 사고 시 모든 사용자 패스워드 강제 재설정 절차 문서화

---

## 10. 함정 모음

### 함정 1 — `email` 만 unique, `lower(email)` 안 함
`a@x.com` 과 `A@x.com` 둘 다 통과. `lower()` 표현식 인덱스 필수.

### 함정 2 — 1차 검증만 하고 DB unique 누락
`existsByEmail` 만으론 race condition. **DB UNIQUE 가 진실의 원천** + `DataIntegrityViolationException` 처리.

### 함정 3 — 평문 패스워드 로그 노출
DTO `toString()` 기본 구현이 password 포함. **record 의 `toString` 직접 오버라이드** 또는 `@JsonIgnore` + custom.

### 함정 4 — 이메일 발송을 트랜잭션 안에서
SMTP 5초 지연 = 트랜잭션 5초. 외부 호출은 **AFTER_COMMIT**.

### 함정 5 — 가입 즉시 ACTIVE
이메일 인증 없이 ACTIVE = 가짜 이메일 가입 가능. **PENDING_VERIFICATION → 인증 → ACTIVE**.

### 함정 6 — 응답에 enumeration 정보
존재 → 409, 미존재 → 201 분리하면 가입자 enumeration. 보안 민감 시 항상 200 + "이메일을 확인하세요" 동일 응답.

### 함정 7 — `name` 의 HTML 미escape
저장은 그대로, **출력 시 escape**. Spring Boot Jackson 기본 escape O.

### 함정 8 — bcrypt 72-byte 컷
긴 passphrase 잘림. argon2id 또는 SHA-256 pre-hash.

### 함정 9 — JPA `ddl-auto: update` 운영
스키마 변경이 무언으로. **`validate` / `none`** + Flyway.

### 함정 10 — Idempotency Key 미사용 (선택)
네트워크 retry 로 중복 가입. 대형 서비스는 `Idempotency-Key` 헤더 + 24h Redis 캐시.

---

## 11. 관련

- [[login-jwt]] — 가입 후 로그인
- [[password-reset]] — 잊은 패스워드
- email-verification (다음) — outbox + verify endpoint
- [[../../../../database/postgresql/security|↗ PG 보안]]
- [[../../../../security/security|↗ security hub]]
- [[api-design|↑ api-design hub]]
