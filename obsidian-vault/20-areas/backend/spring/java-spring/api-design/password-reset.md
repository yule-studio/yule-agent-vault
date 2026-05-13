---
title: "패스워드 리셋 — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - password-reset
---

# 패스워드 리셋 — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Java 21 / 단일 사용 토큰 / 만료 / enumeration 방지 |

**[[api-design|↑ api-design hub]]**

> 전제: [[signup]] · [[login-jwt]]. `users`, `Argon2PasswordEncoder`, `RefreshTokenRepository` 존재.

---

## 1. 무엇을 만드는가

두 단계 흐름:

```
POST /api/v1/auth/password-reset/request    # email 입력 → 이메일로 리셋 링크 전송
POST /api/v1/auth/password-reset/confirm    # 링크의 token + 새 password → 변경
```

### 1.1 요청 / 응답

```http
POST /api/v1/auth/password-reset/request
{ "email": "alice@x.com" }

200 OK
{ "data": { "message": "if the email is registered, a reset link has been sent" } }
```

> **항상 200 + 동일 메시지** — email 등록 여부 enumeration 차단.

```http
POST /api/v1/auth/password-reset/confirm
{ "token": "9f4b3a2c-...-base64url", "newPassword": "N3w-Tr0ub4dor!" }

200 OK
{ "data": { "message": "password updated" } }
```

### 1.2 비기능

- 토큰 **단일 사용** — 한 번 쓰면 무효
- 토큰 만료 **30분**
- **enumeration 차단** — 등록 여부 무관 동일 응답
- 같은 이메일 **1시간 3회** rate limit (이메일 폭탄 방지)
- 리셋 성공 시 해당 사용자의 **모든 RefreshToken 무효화**

---

## 2. 도메인 모델

```java
// src/main/java/com/example/shop/domain/auth/PasswordResetToken.java
public final class PasswordResetToken {

    public enum Status { ACTIVE, USED, REVOKED, EXPIRED }

    private final PasswordResetTokenId id;
    private final UserId userId;
    private final String tokenHash;                // SHA-256 of raw token
    private final Instant issuedAt;
    private final Instant expiresAt;
    private Status status;

    private PasswordResetToken(PasswordResetTokenId id, UserId userId, String hash,
                               Instant issuedAt, Instant expiresAt, Status status) {
        this.id = id; this.userId = userId; this.tokenHash = hash;
        this.issuedAt = issuedAt; this.expiresAt = expiresAt; this.status = status;
    }

    public static PasswordResetToken issue(PasswordResetTokenId id, UserId userId,
                                           String hash, Instant now, Duration ttl) {
        return new PasswordResetToken(id, userId, hash, now, now.plus(ttl), Status.ACTIVE);
    }

    public void consume(Instant now) {
        if (status != Status.ACTIVE) throw new IllegalStateException("not active: " + status);
        if (!now.isBefore(expiresAt)) throw new IllegalStateException("expired");
        this.status = Status.USED;
    }
    public void revoke() { this.status = Status.REVOKED; }
    public boolean isUsable(Instant now) {
        return status == Status.ACTIVE && now.isBefore(expiresAt);
    }

    public PasswordResetTokenId id() { return id; }
    public UserId userId() { return userId; }
    public String tokenHash() { return tokenHash; }
    public Status status() { return status; }

    public static PasswordResetToken reconstitute(PasswordResetTokenId id, UserId userId,
                                                  String hash, Instant issuedAt,
                                                  Instant expiresAt, Status status) {
        return new PasswordResetToken(id, userId, hash, issuedAt, expiresAt, status);
    }
}

public record PasswordResetTokenId(String value) {
    public PasswordResetTokenId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("must be ULID");
    }
}

// src/main/java/com/example/shop/domain/auth/PasswordResetTokenRepository.java
public interface PasswordResetTokenRepository {
    PasswordResetToken save(PasswordResetToken t);
    Optional<PasswordResetToken> findByTokenHash(String hash);
    void revokeAllActiveForUser(UserId userId);
    int countActiveForUserSince(UserId userId, Instant since);
}
```

---

## 3. DB 스키마

```sql
-- V3__create_password_reset_tokens.sql
CREATE TABLE password_reset_tokens (
  id          CHAR(26) PRIMARY KEY,
  user_id     CHAR(26) NOT NULL REFERENCES users(id),
  token_hash  CHAR(64) NOT NULL,
  status      VARCHAR(20) NOT NULL,
  issued_at   TIMESTAMPTZ NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX ux_password_reset_tokens_hash ON password_reset_tokens (token_hash);
CREATE INDEX ix_password_reset_tokens_user ON password_reset_tokens (user_id, status);
```

---

## 4. 구현 — Spring Boot

### 4.1 Request UseCase

```java
// src/main/java/com/example/shop/application/auth/RequestPasswordResetUseCase.java
@Service
public class RequestPasswordResetUseCase {

    private static final Logger log = LoggerFactory.getLogger(RequestPasswordResetUseCase.class);
    private final SecureRandom random = new SecureRandom();

    private final UserRepository users;
    private final PasswordResetTokenRepository tokens;
    private final ApplicationEventPublisher events;
    private final IdGenerator ids;
    private final Clock clock;
    private final Duration ttl;
    private final int ratePerHour;

    public RequestPasswordResetUseCase(
        UserRepository users,
        PasswordResetTokenRepository tokens,
        ApplicationEventPublisher events,
        IdGenerator ids,
        Clock clock,
        @Value("${app.password-reset.ttl}") Duration ttl,
        @Value("${app.password-reset.rate-per-hour}") int ratePerHour
    ) {
        this.users = users; this.tokens = tokens; this.events = events;
        this.ids = ids; this.clock = clock;
        this.ttl = ttl; this.ratePerHour = ratePerHour;
    }

    @Transactional
    public void handle(String email) {
        var normalized = email.trim().toLowerCase(Locale.ROOT);
        var maybeUser = users.findByEmail(new Email(normalized));
        if (maybeUser.isEmpty()) { logSilently(normalized); return; }
        var user = maybeUser.get();

        // 이메일 폭탄 방어
        var now = Instant.now(clock);
        var recent = tokens.countActiveForUserSince(user.id(), now.minus(Duration.ofHours(1)));
        if (recent >= ratePerHour) { logSilently(normalized); return; }

        // 기존 ACTIVE 모두 무효 (한 번에 하나만)
        tokens.revokeAllActiveForUser(user.id());

        var rawBytes = new byte[32];
        random.nextBytes(rawBytes);
        var raw = Base64.getUrlEncoder().withoutPadding().encodeToString(rawBytes);
        var hash = sha256Hex(raw);

        var token = PasswordResetToken.issue(
            new PasswordResetTokenId(ids.next()),
            user.id(), hash, now, ttl
        );
        tokens.save(token);

        events.publishEvent(new PasswordResetRequested(
            user.id(), user.email(), raw, token, now
        ));
    }

    private void logSilently(String email) {
        log.info("password reset requested for non-existent or rate-limited email={}", email);
    }
}

// src/main/java/com/example/shop/domain/auth/events/PasswordResetRequested.java
public record PasswordResetRequested(
    UserId userId, Email email, String rawToken,
    PasswordResetToken token, Instant occurredAt
) implements DomainEvent {}
```

### 4.2 Confirm UseCase

```java
// src/main/java/com/example/shop/application/auth/ConfirmPasswordResetUseCase.java
@Service
public class ConfirmPasswordResetUseCase {

    private final UserRepository users;
    private final PasswordResetTokenRepository tokens;
    private final RefreshTokenRepository refreshTokens;
    private final PasswordEncoder encoder;
    private final PasswordPolicy policy;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    public ConfirmPasswordResetUseCase(
        UserRepository users, PasswordResetTokenRepository tokens,
        RefreshTokenRepository refreshTokens, PasswordEncoder encoder,
        PasswordPolicy policy, ApplicationEventPublisher events, Clock clock
    ) {
        this.users = users; this.tokens = tokens; this.refreshTokens = refreshTokens;
        this.encoder = encoder; this.policy = policy;
        this.events = events; this.clock = clock;
    }

    @Transactional
    public void handle(String rawToken, String newPassword) {
        policy.validate(newPassword);

        var hash = sha256Hex(rawToken);
        var tok = tokens.findByTokenHash(hash)
            .orElseThrow(InvalidPasswordResetTokenException::new);

        var now = Instant.now(clock);
        if (!tok.isUsable(now)) throw new InvalidPasswordResetTokenException();

        var user = users.findById(tok.userId())
            .orElseThrow(InvalidPasswordResetTokenException::new);

        // 새 패스워드가 기존과 같으면 거절
        if (encoder.matches(newPassword, user.currentPasswordHash().value()))
            throw new SamePasswordException();

        user.changePassword(new PasswordHash(encoder.encode(newPassword)));
        users.save(user);

        tok.consume(now);
        tokens.save(tok);

        // 보안: 다른 디바이스 강제 로그아웃
        refreshTokens.revokeAllForUser(user.id());

        user.pullDomainEvents().forEach(events::publishEvent);
        events.publishEvent(new PasswordResetCompleted(user.id(), user.email(), now));
    }
}

public record PasswordResetCompleted(UserId userId, Email email, Instant occurredAt)
    implements DomainEvent {}
```

### 4.3 Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/auth/PasswordResetController.java
@RestController
@RequestMapping("/api/v1/auth/password-reset")
public class PasswordResetController {

    private final RequestPasswordResetUseCase request;
    private final ConfirmPasswordResetUseCase confirm;

    public PasswordResetController(RequestPasswordResetUseCase request,
                                   ConfirmPasswordResetUseCase confirm) {
        this.request = request; this.confirm = confirm;
    }

    @PostMapping("/request")
    public ApiResponse<Map<String, String>> request(@Valid @RequestBody PasswordResetRequest req) {
        request.handle(req.email());
        return ApiResponse.ok(Map.of(
            "message", "if the email is registered, a reset link has been sent"));
    }

    @PostMapping("/confirm")
    public ApiResponse<Map<String, String>> confirm(@Valid @RequestBody PasswordResetConfirmRequest req) {
        confirm.handle(req.token(), req.newPassword());
        return ApiResponse.ok(Map.of("message", "password updated"));
    }
}

public record PasswordResetRequest(@Email @NotBlank String email) {}

public record PasswordResetConfirmRequest(
    @NotBlank String token,
    @NotBlank @Size(min = 8, max = 128) String newPassword
) {
    @Override public String toString() {
        return "PasswordResetConfirmRequest[token=***, newPassword=***]";
    }
}
```

### 4.4 이메일 발송 리스너

```java
// src/main/java/com/example/shop/infrastructure/messaging/PasswordResetEmailListener.java
@Component
public class PasswordResetEmailListener {

    private final EmailOutboxRepository outbox;
    private final IdGenerator ids;
    private final Clock clock;
    private final String resetUrlBase;

    public PasswordResetEmailListener(
        EmailOutboxRepository outbox, IdGenerator ids, Clock clock,
        @Value("${app.web.reset-url-base}") String resetUrlBase
    ) {
        this.outbox = outbox; this.ids = ids; this.clock = clock;
        this.resetUrlBase = resetUrlBase;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(PasswordResetRequested e) {
        var link = resetUrlBase + "?token=" + e.rawToken();
        outbox.enqueue(new EmailOutboxRow(
            ids.next(),
            e.email().value(),
            "password-reset",
            Map.of("link", link, "expiresAt", e.token().status().name()),
            Instant.now(clock)
        ));
    }
}
```

### 4.5 설정

```yaml
# application.yml
app:
  password-reset:
    ttl: PT30M
    rate-per-hour: 3
  web:
    reset-url-base: https://shop.example.com/reset-password
```

### 4.6 보안 헤더 (선택)

```java
// SecurityConfig.chain 에 추가
http.headers(h -> h
    .contentSecurityPolicy(cps -> cps
        .policyDirectives("default-src 'self'; frame-ancestors 'none'"))
    .referrerPolicy(r -> r.policy(ReferrerPolicy.NO_REFERRER))
);
```

---

## 5. 테스트

### 5.1 Request — enumeration 방어

```java
@Test
void request_returns_200_even_for_unknown_email() {
    var res = rest.postForEntity("/api/v1/auth/password-reset/request",
        Map.of("email", "nope@x.com"), Map.class);
    assertThat(res.getStatusCode().value()).isEqualTo(200);
    var msg = (String) ((Map<?, ?>) res.getBody().get("data")).get("message");
    assertThat(msg).contains("if the email is registered");
}
```

### 5.2 Confirm — happy path

```java
@Test
void confirm_with_valid_token_updates_password_and_invalidates_RTs() {
    var f = passwordResetFixture.create("a@x.com", "old-Pass!");
    var rawToken = f.rawToken();

    var res = rest.postForEntity("/api/v1/auth/password-reset/confirm",
        Map.of("token", rawToken, "newPassword", "N3w-pass!"), Map.class);
    assertThat(res.getStatusCode().value()).isEqualTo(200);

    var login = rest.postForEntity("/api/v1/auth/login",
        Map.of("email", "a@x.com", "password", "N3w-pass!"), Map.class);
    assertThat(login.getStatusCode().value()).isEqualTo(200);
}
```

### 5.3 만료 / 단일 사용

```java
@Test
void expired_token_returns_400_or_401() {
    var f = passwordResetFixture.createExpired("b@x.com");
    var res = rest.postForEntity("/api/v1/auth/password-reset/confirm",
        Map.of("token", f.rawToken(), "newPassword", "N3w-pass!"), Map.class);
    assertThat(res.getStatusCode().value()).isIn(400, 401);
}

@Test
void token_can_be_used_only_once() {
    var f = passwordResetFixture.create("c@x.com", "old-Pass!");
    rest.postForEntity("/api/v1/auth/password-reset/confirm",
        Map.of("token", f.rawToken(), "newPassword", "N3w-pass!"), Map.class);
    var res2 = rest.postForEntity("/api/v1/auth/password-reset/confirm",
        Map.of("token", f.rawToken(), "newPassword", "Another-N3w!"), Map.class);
    assertThat(res2.getStatusCode().value()).isIn(400, 401);
}
```

---

## 6. 운영 체크리스트

- [ ] `/request` 응답 시간이 user 존재 여부와 무관하게 비슷한가 (timing)
- [ ] 같은 이메일 1시간 3회 / IP 분당 5회 — rate limit
- [ ] 이메일 outbox 에 raw token 흔적 남지 않게 (발송 후 즉시 삭제)
- [ ] 이메일 링크 `?token=` 이 referrer 로 새는지 (no-referrer 헤더)
- [ ] 만료된 token 청소 스케줄 (매일)
- [ ] 리셋 성공 후 사용자 알림 메일 "비밀번호가 변경되었습니다"
- [ ] 모든 디바이스 강제 로그아웃 (RT revokeAll) 동작 확인
- [ ] PasswordPolicy 재사용 (signup 과 동일)
- [ ] 새 패스워드 = 기존 거절 (UX 선택)

---

## 7. 함정 모음

### 함정 1 — 이메일 존재 여부에 따라 응답 다르게
`200 / 404` 분리 = enumeration. 항상 200 + 동일 메시지.

### 함정 2 — query token 이 로그에 그대로
nginx / 액세스 로그에 `?token=xxx` 남음. 로그에 query 마스킹.

### 함정 3 — 토큰 DB raw 저장
DB 유출 = 모든 reset 링크 사용 가능. **SHA-256 hash 저장**.

### 함정 4 — 토큰 단일 사용 안 함
공격자 가로채면 여러 번 시도. `USED` 상태 전환.

### 함정 5 — 만료 미설정
무한 사용 가능. 30분 권장.

### 함정 6 — 리셋 후 옛 RT 살아 있음
패스워드 변경 후에도 옛 session 접근. `revokeAllForUser`.

### 함정 7 — `/request` 가 동기 SMTP 호출
SMTP 느리면 응답 느림 → 존재 여부 timing 유출. **outbox + AFTER_COMMIT**.

### 함정 8 — 동시 ACTIVE 토큰 여러 개
"한 번에 하나" — 새 요청 시 기존 ACTIVE 모두 무효.

### 함정 9 — Bean Validation 만으로 약한 패스워드 차단 시도
`@Size(min=8)` 만으론 부족. `PasswordPolicy` 로 유출 패스워드 / 사용자명 포함 검사.

### 함정 10 — 링크 클릭으로 즉시 reset
잘못 클릭 = 패스워드 잃음. 반드시 "새 패스워드 입력" 페이지.

---

## 8. 관련

- [[signup]] / [[login-jwt]]
- email-verification (다음) — 유사 토큰 구조
- rate-limiting (다음) — `/request` 보호
- [[../../../../security/security|↗ security hub]]
- [[api-design|↑ api-design hub]]
