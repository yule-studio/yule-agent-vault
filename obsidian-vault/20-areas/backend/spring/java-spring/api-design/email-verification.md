---
title: "이메일 인증 — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - email-verification
---

# 이메일 인증 — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | verification token / outbox / re-send rate / 만료 |

**[[api-design|↑ api-design hub]]**

> 📐 **ORM**: §6 는 JPA Adapter sketch. 3 모드 정책: [[api-design#0.5 ORM 정책]].
> 공통 패턴 — [[../common/response-envelope]] · [[../common/security-config]]
>
> 전제: [[signup]] 으로 user 가 `PENDING_VERIFICATION` 상태로 생성됨.

---

## 1. 무엇을 만드는가

```
POST /api/v1/auth/verify/email/request    # token 발급 + 이메일 발송 (재발송 포함)
GET  /api/v1/auth/verify/email?token=...  # 사용자가 메일의 링크 클릭 시
POST /api/v1/auth/verify/email/confirm    # token + status → ACTIVE 전환
```

### 1.1 비기능

- 토큰 **단일 사용** (USED 상태로 transition) — 가로채기 시 1회 사용
- 토큰 만료 **24시간**
- **재발송 1시간 3회** 제한 (이메일 폭탄 방어)
- 발송은 **outbox + AFTER_COMMIT** (트랜잭션 안에서 SMTP X)
- 인증 성공 시 user.status = `PENDING_VERIFICATION` → `ACTIVE`
- 인증 링크 = `https://shop.example.com/verify?token=...` (프론트가 confirm API 호출)

---

## 2. 도메인 모델

```java
// src/main/java/com/example/shop/domain/auth/EmailVerificationToken.java
public final class EmailVerificationToken {

    public enum Status { ACTIVE, USED, REVOKED, EXPIRED }

    private final EmailVerificationTokenId id;
    private final UserId userId;
    private final String tokenHash;       // SHA-256(raw)
    private final Instant issuedAt;
    private final Instant expiresAt;
    private Status status;

    private EmailVerificationToken(EmailVerificationTokenId id, UserId userId, String hash,
                                   Instant issuedAt, Instant expiresAt, Status status) {
        this.id = id; this.userId = userId; this.tokenHash = hash;
        this.issuedAt = issuedAt; this.expiresAt = expiresAt; this.status = status;
    }

    public static EmailVerificationToken issue(EmailVerificationTokenId id, UserId userId,
                                               String hash, Instant now, Duration ttl) {
        return new EmailVerificationToken(id, userId, hash, now, now.plus(ttl), Status.ACTIVE);
    }

    public void consume(Instant now) {
        if (status != Status.ACTIVE) throw new BusinessException(ResponseCode.INVALID_TOKEN,
            "유효하지 않은 인증 토큰입니다.");
        if (!now.isBefore(expiresAt)) throw new BusinessException(ResponseCode.EXPIRED_AUTH_CODE,
            "인증 코드가 만료되었습니다.");
        status = Status.USED;
    }
    public void revoke() { status = Status.REVOKED; }
    public boolean isUsable(Instant now) { return status == Status.ACTIVE && now.isBefore(expiresAt); }

    public EmailVerificationTokenId id() { return id; }
    public UserId userId() { return userId; }
    public String tokenHash() { return tokenHash; }
    public Status status() { return status; }
}

public record EmailVerificationTokenId(String value) {
    public EmailVerificationTokenId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("ULID 26 chars");
    }
}

// Repository (port)
public interface EmailVerificationTokenRepository {
    EmailVerificationToken save(EmailVerificationToken t);
    Optional<EmailVerificationToken> findByTokenHash(String hash);
    void revokeAllActiveForUser(UserId userId);
    int countActiveForUserSince(UserId userId, Instant since);
}
```

---

## 3. DB 스키마

```sql
-- V60__create_email_verification_tokens.sql
CREATE TABLE email_verification_tokens (
  id          CHAR(26) PRIMARY KEY,
  user_id     CHAR(26) NOT NULL REFERENCES users(id),
  token_hash  CHAR(64) NOT NULL,
  status      VARCHAR(20) NOT NULL,
  issued_at   TIMESTAMPTZ NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX ux_email_verif_tokens_hash ON email_verification_tokens (token_hash);
CREATE INDEX ix_email_verif_tokens_user ON email_verification_tokens (user_id, status);
```

---

## 4. UseCase

### 4.1 RequestEmailVerificationUseCase

```java
// src/main/java/com/example/shop/application/auth/RequestEmailVerificationUseCase.java
@Slf4j
@Service
@RequiredArgsConstructor
public class RequestEmailVerificationUseCase {

    private final UserRepository users;
    private final EmailVerificationTokenRepository tokens;
    private final ApplicationEventPublisher events;
    private final IdGenerator ids;
    private final Clock clock;
    private final SecureRandom random = new SecureRandom();

    @Value("${app.email-verification.ttl:PT24H}") Duration ttl;
    @Value("${app.email-verification.rate-per-hour:3}") int ratePerHour;

    @Transactional
    public void handle(UserId userId) {
        var user = users.findById(userId)
            .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND));

        if (user.isActive()) {
            // 이미 인증됨 — silently 성공 (사용자 혼동 줄임)
            log.info("user {} already active, ignore re-request", userId.value());
            return;
        }

        var now = Instant.now(clock);

        // 1시간 N회 제한
        var recent = tokens.countActiveForUserSince(userId, now.minus(Duration.ofHours(1)));
        if (recent >= ratePerHour) {
            throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                "인증 메일 발송 한도를 초과했습니다. 1시간 후 다시 시도해 주세요.");
        }

        // 기존 ACTIVE 모두 무효
        tokens.revokeAllActiveForUser(userId);

        // 새 토큰
        var rawBytes = new byte[32];
        random.nextBytes(rawBytes);
        var raw = Base64.getUrlEncoder().withoutPadding().encodeToString(rawBytes);
        var hash = sha256Hex(raw);

        var token = EmailVerificationToken.issue(
            new EmailVerificationTokenId(ids.next()),
            userId, hash, now, ttl
        );
        tokens.save(token);

        // 트랜잭션 커밋 후 outbox 적재
        events.publishEvent(new EmailVerificationRequested(
            userId, user.email(), raw, token.id(), now
        ));
    }
}

public record EmailVerificationRequested(
    UserId userId, Email email, String rawToken,
    EmailVerificationTokenId tokenId, Instant occurredAt
) implements DomainEvent {}
```

### 4.2 ConfirmEmailVerificationUseCase

```java
@Service
@RequiredArgsConstructor
public class ConfirmEmailVerificationUseCase {

    private final UserRepository users;
    private final EmailVerificationTokenRepository tokens;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    @Transactional
    public void handle(String rawToken) {
        var hash = sha256Hex(rawToken);
        var tok = tokens.findByTokenHash(hash)
            .orElseThrow(() -> new BusinessException(ResponseCode.INVALID_TOKEN,
                "유효하지 않은 인증 토큰입니다."));

        var now = Instant.now(clock);
        tok.consume(now);                              // 도메인이 만료 / 단일사용 검증
        tokens.save(tok);

        var user = users.findById(tok.userId())
            .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND));
        user.verifyEmail();                            // PENDING_VERIFICATION → ACTIVE
        users.save(user);

        user.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

---

## 5. Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/auth/EmailVerificationController.java
@Tag(name = "이메일 인증", description = "회원가입 후 이메일 인증 API")
@RestController
@RequestMapping("/api/v1/auth/verify/email")
@RequiredArgsConstructor
public class EmailVerificationController {

    private final RequestEmailVerificationUseCase request;
    private final ConfirmEmailVerificationUseCase confirm;

    @Operation(summary = "인증 메일 요청 / 재발송",
               description = "로그인 상태에서 호출. 24시간 유효 토큰 발급 + 이메일 발송.")
    @PostMapping("/request")
    public ResponseEntity<CommonResponse<Void>> request(Authentication auth) {
        request.handle(new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            "인증 메일을 발송했습니다."));
    }

    @Operation(summary = "이메일 인증 confirm",
               description = "메일의 링크 클릭 시 프론트가 호출. 토큰 검증 + user ACTIVE 전환.")
    @SecurityRequirements()                          // 비인증
    @PostMapping("/confirm")
    public ResponseEntity<CommonResponse<Void>> confirm(@Valid @RequestBody EmailConfirmRequest req) {
        confirm.handle(req.token());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            "이메일 인증이 완료되었습니다."));
    }
}

public record EmailConfirmRequest(@NotBlank String token) {
    @Override public String toString() { return "EmailConfirmRequest[token=***]"; }
}
```

---

## 6. 이메일 발송 listener (outbox)

```java
// src/main/java/com/example/shop/infrastructure/messaging/EmailVerificationListener.java
@Component
@RequiredArgsConstructor
public class EmailVerificationListener {

    private final EmailOutboxRepository outbox;
    private final IdGenerator ids;
    private final Clock clock;

    @Value("${app.web.verify-email-base}") String verifyUrlBase;
    @Value("${app.web.verify-email-ttl-display:24시간}") String ttlDisplay;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(EmailVerificationRequested event) {
        var link = verifyUrlBase + "?token=" + event.rawToken();
        outbox.enqueue(new EmailOutboxRow(
            ids.next(),
            event.email().value(),
            "email-verification",
            Map.of(
                "link", link,
                "ttl", ttlDisplay
            ),
            Instant.now(clock)
        ));
    }
}
```

```yaml
# application.yml
app:
  email-verification:
    ttl: PT24H
    rate-per-hour: 3
  web:
    verify-email-base: https://shop.example.com/verify-email
```

---

## 7. SecurityConfig 갱신

`/api/v1/auth/verify/email/confirm` 은 비인증 접근. `request` 는 인증 필요.

```java
// SecurityConfig.filterChain 의 authorizeHttpRequests 에
.requestMatchers(
    "/api/v1/auth/verify/email/confirm"            // confirm 만 공개
).permitAll()
// request 는 anyRequest().authenticated() 에 포함
```

---

## 8. 테스트

```java
@Test
void confirm_activates_user() {
    var user = userFixture.createPending("a@x.com");
    var token = verifyFixture.issue(user.id());

    var res = rest.postForEntity("/api/v1/auth/verify/email/confirm",
        Map.of("token", token.raw()), Map.class);
    assertThat(res.getStatusCode().value()).isEqualTo(200);

    var reloaded = userFixture.reload(user.id());
    assertThat(reloaded.status()).isEqualTo(UserStatus.ACTIVE);
}

@Test
void request_more_than_rate_returns_429() {
    var user = userFixture.createPending("a@x.com");
    var auth = userFixture.accessToken(user);
    for (int i = 0; i < 3; i++) {
        var res = rest.exchange("/api/v1/auth/verify/email/request",
            HttpMethod.POST, new HttpEntity<>(null, headers(auth)), Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(200);
    }
    var fourth = rest.exchange("/api/v1/auth/verify/email/request",
        HttpMethod.POST, new HttpEntity<>(null, headers(auth)), Map.class);
    assertThat(fourth.getStatusCode().value()).isEqualTo(429);
}
```

---

## 9. 운영 체크리스트

- [ ] `verify-email-base` 가 HTTPS 도메인
- [ ] 이메일 outbox 워커 라이브니스 + 대기 큐 길이 알람
- [ ] referrer 헤더로 토큰 새지 않게 (no-referrer policy)
- [ ] 만료 토큰 매일 cleanup
- [ ] confirm endpoint 에 rate limit (IP 차원)
- [ ] 이미 ACTIVE 인 user 가 request 호출 시 silently 성공 (혼동 방지)

---

## 10. 함정 모음

### 함정 1 — 토큰을 raw 로 DB 저장
DB 유출 = 모든 인증 링크 사용 가능. **SHA-256 hash**.

### 함정 2 — 만료 미설정
영구 유효 토큰 = 보안 사고. 24시간 권장.

### 함정 3 — query token 로그 노출
nginx access log 에 `?token=...` 남음. log 필터링 또는 path 만 기록.

### 함정 4 — 동기 SMTP 호출
회원가입 트랜잭션이 SMTP 응답 기다림 → 락 hold. **outbox + AFTER_COMMIT**.

### 함정 5 — 재발송 rate 없음
이메일 폭탄 / 비용 폭증. 1시간 N회 제한.

### 함정 6 — 인증 완료 후 자동 로그인
사용자가 다른 브라우저에서 confirm 클릭하면 거기서 로그인됨? — 보안 위험. **인증만 처리, 로그인 별도**.

### 함정 7 — 만료 시 새 토큰 발급 자동
사용자가 만료 링크 클릭 = 친절히 자동 재발송? — UX OK 지만 abuse 위험. 사용자가 명시적으로 재요청.

### 함정 8 — verification token = password reset token 공유 X
다른 종류 (verify / reset / 2fa-recovery). **각자 분리**.

---

## 11. 관련

- [[signup]] — user 생성 (PENDING_VERIFICATION)
- [[password-reset]] — 유사 토큰 구조
- [[../common/response-envelope]] · [[../common/security-config]]
- rate-limiting (다음) — request endpoint 보호
- [[api-design|↑ api-design hub]]
