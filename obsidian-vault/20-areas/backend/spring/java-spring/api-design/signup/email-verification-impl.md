---
title: "auth §10 — 이메일 인증 구현 (AWS SES 권장)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - email-verification
---

# auth §10 — 이메일 인증 구현 (AWS SES 권장)

**[[signup|↑ hub]]**  ·  ← [[signup-impl]]  ·  → [[phone-verification-impl]]

> 가입 후 `PENDING_VERIFICATION` user 가 이메일의 링크를 클릭해 `ACTIVE` 로 전이.
> 도구 선택: [[design-decisions#5 이메일 발송 도구]] 참고.

---

## 1. API spec

```
POST /api/v1/auth/verify/email/request    # 인증 메일 발송 (로그인 후)
POST /api/v1/auth/verify/email/confirm    # 메일 링크의 토큰 검증 (비인증)
```

### 1.1 Request — 인증 메일 발송 (로그인 후)

```http
POST /api/v1/auth/verify/email/request
Authorization: Bearer ...

200 OK
{ "code": "OK_001", "message": "인증 메일을 발송했습니다." }
```

### 1.2 Confirm — 메일 링크의 토큰 검증

```http
POST /api/v1/auth/verify/email/confirm
Content-Type: application/json
{ "token": "9f4b3a2c-...-base64url" }

200 OK
{ "code": "OK_001", "message": "이메일 인증 완료" }
```

---

## 2. 비기능

- 토큰 **단일 사용** (USED 상태)
- 토큰 만료 **24시간**
- 같은 user 의 재발송 **1시간 3회** 제한
- 발송은 **outbox + AFTER_COMMIT** (트랜잭션 안 SMTP X)
- 이미 ACTIVE 인 user 가 request 호출 시 — silently 성공 (혼동 방지)
- 인증 링크: `https://shop.example.com/verify-email?token=...` (프론트가 confirm API 호출)

---

## 3. 도메인 / DB

```java
public final class EmailVerificationToken {

    public enum Status { ACTIVE, USED, REVOKED, EXPIRED }

    private final EmailVerificationTokenId id;
    private final UserId userId;
    private final String tokenHash;
    private final Instant issuedAt;
    private final Instant expiresAt;
    private Status status;

    public static EmailVerificationToken issue(EmailVerificationTokenId id, UserId userId,
                                                String hash, Instant now, Duration ttl) {
        return new EmailVerificationToken(id, userId, hash, now, now.plus(ttl), Status.ACTIVE);
    }

    public void consume(Instant now) {
        if (status != Status.ACTIVE) throw new BusinessException(ResponseCode.INVALID_TOKEN);
        if (!now.isBefore(expiresAt)) throw new BusinessException(ResponseCode.EXPIRED_AUTH_CODE);
        status = Status.USED;
    }
}

public interface EmailVerificationTokenRepository {
    EmailVerificationToken save(EmailVerificationToken t);
    Optional<EmailVerificationToken> findByTokenHash(String hash);
    void revokeAllActiveForUser(UserId userId);
    int countActiveForUserSince(UserId userId, Instant since);
}
```

```sql
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

## 4. UseCase — Request

```java
@Service
@Slf4j
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
            log.info("user {} already active, ignore re-request", userId.value());
            return;
        }

        var now = Instant.now(clock);
        var recent = tokens.countActiveForUserSince(userId, now.minus(Duration.ofHours(1)));
        if (recent >= ratePerHour)
            throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                "인증 메일 발송 한도 초과. 1시간 후 다시 시도해 주세요.");

        tokens.revokeAllActiveForUser(userId);

        var rawBytes = new byte[32];
        random.nextBytes(rawBytes);
        var raw = Base64.getUrlEncoder().withoutPadding().encodeToString(rawBytes);
        var hash = sha256Hex(raw);

        var token = EmailVerificationToken.issue(
            new EmailVerificationTokenId(ids.next()),
            userId, hash, now, ttl
        );
        tokens.save(token);

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

---

## 5. UseCase — Confirm

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
            .orElseThrow(() -> new BusinessException(ResponseCode.INVALID_TOKEN));

        var now = Instant.now(clock);
        tok.consume(now);                      // 만료 / 단일 사용 검증
        tokens.save(tok);

        var user = users.findById(tok.userId())
            .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND));
        user.verifyEmail();                    // PENDING_VERIFICATION → ACTIVE
        users.save(user);

        user.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

---

## 6. AWS SES Email Client

```kotlin
// build.gradle.kts
implementation("software.amazon.awssdk:ses:2.25.50")
```

```yaml
app:
  email:
    provider: aws-ses                      # 또는 sendgrid / smtp
    from: noreply@shop.example.com
    aws-region: ap-northeast-2

aws:
  ses:
    access-key-id: ${AWS_ACCESS_KEY_ID:}
    secret-access-key: ${AWS_SECRET_ACCESS_KEY:}
```

```java
// infrastructure/external/email/AwsSesEmailClient.java
@Component
@RequiredArgsConstructor
@Slf4j
public class AwsSesEmailClient implements EmailClient {

    private final SesClient ses;
    private final TemplateEngine templates;       // Thymeleaf / Freemarker
    @Value("${app.email.from}") String fromAddress;

    @CircuitBreaker(name = "email-ses", fallbackMethod = "fallback")
    @Retry(name = "email-ses")
    @Override
    public EmailResult send(String to, String templateName, Map<String, Object> payload) {
        var html = templates.process(templateName, new Context(Locale.KOREAN, payload));
        var subject = subjectOf(templateName);

        var request = SendEmailRequest.builder()
            .source(fromAddress)
            .destination(Destination.builder().toAddresses(to).build())
            .message(Message.builder()
                .subject(Content.builder().data(subject).charset("UTF-8").build())
                .body(Body.builder()
                    .html(Content.builder().data(html).charset("UTF-8").build())
                    .build())
                .build())
            .build();

        try {
            var response = ses.sendEmail(request);
            return new EmailResult(true, response.messageId(), null);
        } catch (SesException e) {
            log.warn("SES send failed: to={}, error={}", to, e.awsErrorDetails().errorMessage());
            return new EmailResult(false, null, e.awsErrorDetails().errorCode());
        }
    }

    private EmailResult fallback(String to, String tpl, Map<String, Object> p, Throwable t) {
        return new EmailResult(false, null, "SES temporarily unavailable: " + t.getMessage());
    }

    private String subjectOf(String tpl) {
        return switch (tpl) {
            case "email-verification" -> "[Shop] 이메일 인증을 완료해 주세요";
            case "password-reset" -> "[Shop] 패스워드 재설정";
            default -> "[Shop] 알림";
        };
    }
}

public interface EmailClient {
    EmailResult send(String to, String templateName, Map<String, Object> payload);
    record EmailResult(boolean ok, String externalMessageId, String errorCode) {}
}
```

---

## 7. Outbox Listener + Worker

```java
@Component
@RequiredArgsConstructor
public class EmailVerificationOutboxListener {

    private final EmailOutboxRepository outbox;
    private final IdGenerator ids;
    private final Clock clock;
    @Value("${app.web.verify-email-base}") String verifyUrlBase;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(EmailVerificationRequested event) {
        var link = verifyUrlBase + "?token=" + event.rawToken();
        outbox.enqueue(new EmailOutboxRow(
            ids.next(),
            event.email().value(),
            "email-verification",
            Map.of("link", link, "expiresAt", event.tokenId().value()),
            Instant.now(clock)
        ));
    }
}
```

```java
// Worker — outbox 의 PENDING row 를 polling, SES 호출
@Component
@RequiredArgsConstructor
public class EmailOutboxWorker {

    private final EmailOutboxRepository outbox;
    private final EmailClient client;

    @Scheduled(fixedDelay = 1000)
    @SchedulerLock(name = "emailOutboxWorker", lockAtMostFor = "5m")
    public void process() {
        outbox.findPending(50).forEach(this::sendOne);
    }

    @Transactional
    public void sendOne(EmailOutboxRow row) {
        var result = client.send(row.toEmail(), row.template(), row.payload());
        if (result.ok()) {
            outbox.markSent(row.id(), result.externalMessageId(), Instant.now());
        } else {
            outbox.recordFailure(row.id(), result.errorCode());
        }
    }
}
```

---

## 8. Controller

```java
@Tag(name = "이메일 인증")
@RestController
@RequestMapping("/api/v1/auth/verify/email")
@RequiredArgsConstructor
public class EmailVerificationController {

    private final RequestEmailVerificationUseCase request;
    private final ConfirmEmailVerificationUseCase confirm;

    @Operation(summary = "인증 메일 요청 / 재발송")
    @PostMapping("/request")
    public ResponseEntity<CommonResponse<Void>> requestVerify(Authentication auth) {
        request.handle(new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            "인증 메일을 발송했습니다."));
    }

    @Operation(summary = "이메일 인증 confirm")
    @SecurityRequirements()
    @PostMapping("/confirm")
    public ResponseEntity<CommonResponse<Void>> confirmVerify(
        @Valid @RequestBody EmailConfirmDto req
    ) {
        confirm.handle(req.token());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            "이메일 인증이 완료되었습니다."));
    }
}

public record EmailConfirmDto(@NotBlank String token) {
    @Override public String toString() { return "EmailConfirmDto[token=***]"; }
}
```

---

## 9. SES sandbox 해제 / DNS 설정

운영 전 필수:
- **SES sandbox 해제** — AWS 콘솔에서 production access 요청 (1~2일)
- **SPF** DNS: `v=spf1 include:amazonses.com -all`
- **DKIM** — SES 가 발급한 CNAME 3개 등록
- **DMARC** — `v=DMARC1; p=quarantine; rua=mailto:dmarc@shop.example.com`
- **From 도메인 검증** — `noreply@shop.example.com`

→ 설정 안 하면 Gmail / Naver 가 spam 폴더 직행. **반드시 SPF + DKIM + DMARC**.

---

## 10. SendGrid 대안 (간단)

AWS SES 가 어렵거나 배달율이 critical 한 경우:

```kotlin
implementation("com.sendgrid:sendgrid-java:4.10.2")
```

```java
@Component
public class SendGridEmailClient implements EmailClient {
    private final SendGrid sg;
    @Override public EmailResult send(String to, String tpl, Map<String, Object> payload) {
        // Mail + Email + Content 빌더 → sg.api(Request)
    }
}
```

> SendGrid 는 transactional 메일 배달율 가장 좋음. 단 SES 보다 비쌈 ($20+ vs $0.10).

---

## 11. 함정 모음

### 함정 1 — SMTP 직접 호출 (Gmail / 자체)
배달율 ↓ + IP 등록 X = spam 직행. **SES / SendGrid / Postmark**.

### 함정 2 — SPF / DKIM 없음
spam 폴더 직행. **DNS 등록 필수**.

### 함정 3 — 동기 SMTP
회원가입 트랜잭션이 SMTP 응답 기다림. **outbox + AFTER_COMMIT**.

### 함정 4 — 토큰 raw 저장
DB 유출. SHA-256 hash.

### 함정 5 — 만료 미설정
영구 유효 토큰 = 보안 사고. 24시간 권장.

### 함정 6 — query token 이 로그에 노출
nginx access log + 외부 분석 도구 (DataDog 등). **path masking** 또는 path 만.

### 함정 7 — 재발송 무제한
이메일 폭탄. 1시간 3회.

### 함정 8 — 메일 본문에 markdown 만
일부 이메일 클라가 markdown 인식 X. **HTML + text** 둘 다.

### 함정 9 — 외부 IP / 브라우저 fingerprint 검증 X
링크 클릭한 IP 가 가입 IP 와 달라도 OK (legit 사용). 단 의심 시 알람.

### 함정 10 — bounce / complaint webhook 무시
spam 신고 받은 주소에 계속 발송 → 발신 도메인 reputation 망가짐. **SES bounce webhook → 차단 리스트**.

---

## 12. 운영 체크리스트

- [ ] SES sandbox 해제
- [ ] SPF + DKIM + DMARC DNS 등록
- [ ] bounce / complaint webhook 처리
- [ ] outbox 워커 라이브니스
- [ ] 발송 성공율 메트릭
- [ ] 토큰 SHA-256 hash
- [ ] 24h TTL
- [ ] Referrer-Policy: no-referrer
- [ ] 1h 3회 rate limit

---

## 13. 관련

- [[signup|↑ hub]]
- [[signup-impl]] — 이전 (§9) — UserRegistered event 가 트리거
- [[phone-verification-impl]] — 다음 (§11)
- [[password-reset-impl]] — 유사 토큰 패턴
- [[design-decisions#5 이메일 발송 도구]]
