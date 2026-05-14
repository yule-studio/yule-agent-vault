---
title: "signup §6 — 구현 (Java + Spring Boot)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:44:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - implementation
---

# signup §6 — 구현 (Java + Spring Boot)

**[[signup|↑ signup hub]]**  ·  ← [[security]]  ·  → [[transactions]]

---

## 1. 패키지 구조

[[architecture#5 패키지 트리]] 참고. 본 §6 는 각 파일의 코드.

---

## 2. 도메인 — Value Objects

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

// src/main/java/com/example/shop/domain/user/UserStatus.java
public enum UserStatus { PENDING_VERIFICATION, ACTIVE, SUSPENDED, DELETED }
```

---

## 3. 도메인 — User Aggregate

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
        if (status == UserStatus.DELETED) throw new IllegalStateException("deleted");
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
        var copy = List.copyOf(events); events.clear(); return copy;
    }

    public UserId id() { return id; }
    public Email email() { return email; }
    public String name() { return name; }
    public UserStatus status() { return status; }
    public Instant createdAt() { return createdAt; }
    public PasswordHash currentPasswordHash() { return passwordHash; }
    public boolean isActive() { return status == UserStatus.ACTIVE; }

    public static User reconstitute(UserId id, Email email, PasswordHash hash,
                                    String name, UserStatus status, Instant createdAt) {
        return new User(id, email, hash, name, status, createdAt);
    }
}
```

---

## 4. 도메인 — Events + Port

```java
// domain/common/DomainEvent.java
public sealed interface DomainEvent
    permits UserRegistered, UserEmailVerified, UserPasswordChanged { Instant occurredAt(); }

// domain/user/events/UserRegistered.java
public record UserRegistered(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserEmailVerified(UserId userId, Email email, Instant occurredAt) implements DomainEvent {}
public record UserPasswordChanged(UserId userId, Instant occurredAt) implements DomainEvent {}

// domain/user/UserRepository.java  (port)
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    boolean existsByEmail(Email email);
    User save(User user);
    Optional<User> findById(UserId id);
}

// domain/user/PasswordEncoder.java  (port)
public interface PasswordEncoder {
    String encode(String plain);
    boolean matches(String plain, String hash);
}

// domain/user/exceptions/EmailAlreadyExistsException.java
public class EmailAlreadyExistsException extends BusinessException {
    public EmailAlreadyExistsException(Email email) {
        super(ResponseCode.DUPLICATE_DATA, "이미 가입된 이메일입니다: " + email.value());
    }
}
```

---

## 5. Application — Command + UseCase + Policy

```java
// application/user/SignupCommand.java
public record SignupCommand(
    String email,
    String rawPassword,
    String name,
    boolean termsAgreed,
    boolean marketingAgreed
) {}

// application/user/PasswordPolicy.java
@Component
@RequiredArgsConstructor
public class PasswordPolicy {

    private final PwnedPasswordChecker pwned;       // optional bean — null OK

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

```java
// application/user/SignupUseCase.java
@Service
@Slf4j
@RequiredArgsConstructor
public class SignupUseCase {

    private final UserRepository users;
    private final PasswordEncoder passwordEncoder;
    private final PasswordPolicy passwordPolicy;
    private final UserTermsConsentService termsConsent;
    private final IdGenerator idGenerator;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public User handle(SignupCommand cmd) {
        if (!cmd.termsAgreed())
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT, "약관 동의 필요");
        passwordPolicy.validate(cmd.rawPassword());

        var email = new Email(cmd.email().trim().toLowerCase(Locale.ROOT));

        if (users.existsByEmail(email))
            throw new EmailAlreadyExistsException(email);

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

        // 약관 동의 저장
        termsConsent.save(saved.id(), cmd.termsAgreed(), cmd.marketingAgreed(),
                          Instant.now(clock));

        // 도메인 이벤트 발행 — listener 가 후속 처리
        saved.pullDomainEvents().forEach(events::publishEvent);
        return saved;
    }
}
```

---

## 6. Application — UserTermsConsentService

```java
@Service
@RequiredArgsConstructor
public class UserTermsConsentService {

    private final TermsRepository terms;
    private final UserTermsConsentHistoryRepository consents;
    private final IdGenerator ids;

    @Transactional
    public void save(UserId userId, boolean termsAgreed, boolean marketingAgreed, Instant now) {
        var active = terms.findActive();
        for (var t : active) {
            boolean consent = t.isRequired()
                ? true                                  // 필수 약관 — 가입 자체가 동의 의미
                : "marketing".equals(t.getCode()) ? marketingAgreed : false;
            consents.save(new UserTermsConsentHistoryRow(
                ids.next(), userId.value(), t.getId(),
                consent ? "Y" : "N", now
            ));
        }
    }
}
```

---

## 7. Infrastructure — JPA Adapter

```java
// infrastructure/persistence/jpa/user/UserJpaEntity.java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(nullable = false, length = 254) private String email;
    @Column(name = "password_hash", nullable = false, length = 255) private String passwordHash;
    @Column(nullable = false, length = 100) private String name;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30) private UserStatus status;
    @Version @Column(nullable = false) private long version;
    @Column(name = "created_at", nullable = false, updatable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;

    protected UserJpaEntity() {}
    public UserJpaEntity(String id, String email, String passwordHash, String name,
                         UserStatus status, Instant createdAt, Instant updatedAt) {
        this.id = id; this.email = email; this.passwordHash = passwordHash;
        this.name = name; this.status = status;
        this.createdAt = createdAt; this.updatedAt = updatedAt;
    }
    // getters / package-private setters
    void apply(User u, Instant now) {
        this.passwordHash = u.currentPasswordHash().value();
        this.status = u.status();
        this.updatedAt = now;
    }
}

// infrastructure/persistence/jpa/user/UserJpaRepository.java
public interface UserJpaRepository extends JpaRepository<UserJpaEntity, String> {
    @Query("select case when count(u) > 0 then true else false end " +
           "from UserJpaEntity u where lower(u.email) = lower(:email)")
    boolean existsByEmailIgnoreCase(@Param("email") String email);

    @Query("from UserJpaEntity u where lower(u.email) = lower(:email)")
    Optional<UserJpaEntity> findByEmailIgnoreCase(@Param("email") String email);
}

// infrastructure/persistence/jpa/user/JpaUserRepositoryAdapter.java
@Repository
@RequiredArgsConstructor
public class JpaUserRepositoryAdapter implements UserRepository {

    private final UserJpaRepository spring;
    private final Clock clock;

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
                user.status(), user.createdAt(), now));
        return toDomain(spring.save(entity));
    }

    private User toDomain(UserJpaEntity e) {
        return User.reconstitute(
            new UserId(e.getId()), new Email(e.getEmail()),
            new PasswordHash(e.getPasswordHash()), e.getName(),
            e.getStatus(), e.getCreatedAt()
        );
    }
}
```

> MyBatis Adapter 변형은 [[../../database/mybatis#10.5.1]] 참고.

---

## 8. Infrastructure — Argon2PasswordEncoder

[[security#4 알고리즘 선정]] 참고. 그 코드 그대로.

---

## 9. Presentation — DTO + Controller + Exception

```java
// presentation/api/v1/auth/SignupRequest.java
public record SignupRequest(
    @Email @NotBlank @Size(max = 254) String email,
    @NotBlank @Size(min = 8, max = 128) String password,
    @NotBlank @Size(max = 100) String name,
    @AssertTrue(message = "must agree to terms") boolean termsAgreed,
    boolean marketingAgreed
) {
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s, termsAgreed=%s]"
            .formatted(email, name, termsAgreed);
    }
}

// presentation/api/v1/auth/SignupResponse.java
public record SignupResponse(String userId, String email, String status, Instant createdAt) {}

// presentation/api/v1/auth/SignupController.java
@Tag(name = "사용자 회원가입")
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class SignupController {

    private final SignupUseCase signupUseCase;

    @Operation(summary = "회원가입")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "(OK_001) 회원가입 성공"),
        @ApiResponse(responseCode = "409", description = "(BADREQ_004) 이미 가입된 이메일"),
        @ApiResponse(responseCode = "422", description = "(BADREQ_002) 약한 패스워드 / 입력 형식")
    })
    @SecurityRequirements()
    @PostMapping("/signup")
    public ResponseEntity<CommonResponse<SignupResponse>> signup(
        @Valid @RequestBody SignupRequest req
    ) {
        var user = signupUseCase.handle(toCommand(req));
        var body = CommonResponse.success(ResponseCode.OK,
            new SignupResponse(
                user.id().value(),
                user.email().value(),
                user.status().name(),
                user.createdAt()
            ),
            "회원가입 성공"
        );
        return ResponseEntity.ok(body);
    }

    private static SignupCommand toCommand(SignupRequest r) {
        return new SignupCommand(r.email(), r.password(), r.name(),
                                 r.termsAgreed(), r.marketingAgreed());
    }
}
```

→ Exception handler 는 [[../../common/response-envelope#6]] 의 표준 `ApiExceptionHandler` 그대로. `BusinessException` / `EmailAlreadyExistsException` / `MethodArgumentNotValidException` / `DataIntegrityViolationException` 모두 매핑.

---

## 10. Infrastructure — UserRegisteredEmailListener

```java
// infrastructure/messaging/UserRegisteredEmailListener.java
@Component
@RequiredArgsConstructor
public class UserRegisteredEmailListener {

    private final EmailOutboxRepository outbox;
    private final IdGenerator ids;
    private final Clock clock;

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

→ 트랜잭션 커밋 후에만 outbox 적재. 실제 메일 발송은 별도 워커 ([[../email-verification]]).

---

## 11. Config — Beans

```java
// config/CommonConfig.java
@Configuration
public class CommonConfig {
    @Bean public Clock clock() { return Clock.systemUTC(); }
    @Bean public IdGenerator idGenerator() { return () -> ULID.random(); }
}
```

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}             # TODO(secret): vault 주입
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_batch_fetch_size: 100
  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    com.example.shop: INFO

management:
  endpoint:
    configprops:
      show-values: never
```

---

## 12. 관련

- [[signup|↑ signup hub]]
- [[security]] — 이전 (§5)
- [[transactions]] — 다음 (§7)
- [[architecture#5 패키지 트리]]
- [[../../common/response-envelope]] — `CommonResponse` / `ResponseCode` / `ApiExceptionHandler`
- [[../../common/security-config]] — SecurityConfig
- [[../../database/jpa#11.5.1]] — JPA Adapter 깊이
- [[../../database/mybatis#10.5.1]] — MyBatis 변형
