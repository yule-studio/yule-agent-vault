---
title: "auth §12 — 로그인 구현 (JWT access + refresh)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - login
  - jwt
---

# auth §12 — 로그인 구현 (JWT access + refresh)

**[[signup|↑ hub]]**  ·  ← [[phone-verification-impl]]  ·  → [[token-refresh-impl]]

> 가입된 user 가 이메일·패스워드로 로그인 → JWT access + opaque refresh 발급.
> 도구 선택: [[design-decisions#4 토큰 모델]] 참고.

---

## 1. API spec

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "alice@x.com", "password": "Tr0ub4dor!" }
```

```http
200 OK
{
  "code": "OK_001",
  "status": "OK",
  "message": "로그인 성공",
  "result": {
    "accessToken":  "eyJhbGciOi...",
    "refreshToken": "9f4b3a2c-...-base64url",
    "tokenType":    "Bearer",
    "expiresIn":    900
  }
}
```

실패:
```json
401 INVALID_CREDENTIALS — "invalid email or password"
403 ACCOUNT_NOT_ACTIVE   — "활성 계정이 아닙니다 (PENDING_VERIFICATION)"
401 ACCOUNT_LOCKED       — "5회 로그인 실패 — 15분 후 다시 시도"
```

---

## 2. 비기능

- access TTL **15 분** / refresh TTL **14 일**
- refresh **rotation** (사용 시 새 RT 발급, 옛 RT → ROTATED)
- 동시 다중 세션 허용
- 실패 응답 시간 = 성공과 비슷 (timing 균일화 — dummy hash 비교)
- 5회 연속 실패 시 lock (per IP+email, 15분)
- enumeration 차단 — "invalid email or password" 통일 메시지

---

## 3. 도메인 — RefreshToken Aggregate

```java
// domain/auth/RefreshToken.java
public final class RefreshToken {

    public enum Status { ACTIVE, ROTATED, REVOKED, EXPIRED }

    private final RefreshTokenId id;                  // jti
    private final UserId userId;
    private final String tokenHash;                   // SHA-256(raw)
    private final String deviceFingerprint;
    private final Instant issuedAt;
    private final Instant expiresAt;
    private Status status;
    private RefreshTokenId rotatedToId;

    public static RefreshToken issue(RefreshTokenId id, UserId userId, String tokenHash,
                                     String device, Instant now, Duration ttl) {
        return new RefreshToken(id, userId, tokenHash, device, now,
                                now.plus(ttl), Status.ACTIVE, null);
    }

    public void rotate(RefreshTokenId newId, Instant now) {
        if (status != Status.ACTIVE) throw new IllegalStateException("not active: " + status);
        if (!now.isBefore(expiresAt)) throw new IllegalStateException("expired");
        this.status = Status.ROTATED;
        this.rotatedToId = newId;
    }
    public void revoke() { this.status = Status.REVOKED; }
    public boolean isUsable(Instant now) {
        return status == Status.ACTIVE && now.isBefore(expiresAt);
    }
    // getters
}

public record RefreshTokenId(String value) {
    public RefreshTokenId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("ULID required");
    }
}
```

```java
public interface RefreshTokenRepository {
    RefreshToken save(RefreshToken token);
    Optional<RefreshToken> findById(RefreshTokenId id);
    Optional<RefreshToken> findByTokenHash(String hash);
    void revokeAllForUser(UserId userId);
}
```

---

## 4. JwtTokenProvider — production 표준

→ [[../../common/security-config#7 JwtTokenProvider]] 의 표준 그대로 사용.

핵심:
- HS256 (`Keys.hmacShaKeyFor(Base64.getDecoder().decode(secret))`)
- claim: `sub=email`, `id`, `role`, `sid` (session UUID), `token_type` (ACCESS/REFRESH), `jti`
- access: 위 claim 다 / refresh: jti 만 (PII X)
- `requireIssuer` + `clockSkewSeconds(30)`

---

## 5. UseCase — LoginUseCase

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class LoginUseCase {

    private final UserRepository users;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider jwt;
    private final RefreshTokenService rtService;
    private final LoginAttemptLimiter loginLimiter;
    private final Clock clock;

    // argon2id pre-computed dummy. 실제 hash 형태여야 timing 비슷.
    private static final String DUMMY_HASH =
        "$argon2id$v=19$m=65536,t=3,p=4$AAAAAAAAAAAAAAAAAAAAAA$AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";

    @Transactional
    public LoginResult login(String email, String rawPassword, String device, String ip) {
        var normalized = email.trim().toLowerCase(Locale.ROOT);

        // 1. Rate limit 검사 (IP+email)
        loginLimiter.checkLimit(ip, normalized);

        // 2. user lookup
        var maybeUser = users.findByEmail(new Email(normalized));
        if (maybeUser.isEmpty()) {
            // timing 균일화 — 사용자 없어도 dummy hash 비교
            passwordEncoder.matches(rawPassword, DUMMY_HASH);
            loginLimiter.recordFailure(ip, normalized);
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "invalid email or password");
        }

        var user = maybeUser.get();

        // 3. password 검증
        if (!passwordEncoder.matches(rawPassword, user.currentPasswordHash().value())) {
            loginLimiter.recordFailure(ip, normalized);
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "invalid email or password");
        }

        // 4. status 검증
        if (!user.isActive()) {
            throw new BusinessException(ResponseCode.FORBIDDEN,
                "활성 계정이 아닙니다 (" + user.status().name() + ")");
        }

        // 5. 성공 — rate limit reset
        loginLimiter.recordSuccess(ip, normalized);

        // 6. 토큰 발급
        var sessionId = UUID.randomUUID().toString();
        var access = jwt.generateAccessToken(user.email().value(), user.id().value(),
                                             user.role(), sessionId);
        var rt = rtService.issue(user.id(), device, ip);

        log.info("login success — user={}, ip={}", user.id().value(), ip);
        return new LoginResult(access, rt.raw(), rt.expiresAt());
    }
}

public record LoginResult(String accessToken, String refreshToken, Instant refreshExpiresAt) {}
```

---

## 6. RefreshTokenService — 발급 / Rotation / Revoke

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository tokens;
    private final IdGenerator ids;
    private final Clock clock;
    private final SecureRandom random = new SecureRandom();

    @Value("${jwt.refresh-days:14}") int refreshDays;

    public IssuedRefreshToken issue(UserId userId, String device, String ip) {
        var rawBytes = new byte[32];
        random.nextBytes(rawBytes);
        var raw = Base64.getUrlEncoder().withoutPadding().encodeToString(rawBytes);
        var hash = sha256Hex(raw);

        var token = RefreshToken.issue(
            new RefreshTokenId(ids.next()), userId, hash, device,
            Instant.now(clock), Duration.ofDays(refreshDays)
        );
        tokens.save(token);
        return new IssuedRefreshToken(raw, token.id(), token.expiresAt(), userId);
    }

    @Transactional
    public IssuedRefreshToken rotate(String rawIncoming, String device, String ip) {
        var hash = sha256Hex(rawIncoming);
        var current = tokens.findByTokenHash(hash)
            .orElseThrow(() -> new BusinessException(ResponseCode.INVALID_TOKEN));

        var now = Instant.now(clock);
        switch (current.status()) {
            case ROTATED -> {
                // 도난 의심 — 해당 user 의 모든 RT REVOKE
                tokens.revokeAllForUser(current.userId());
                throw new BusinessException(ResponseCode.UNAUTHORIZED,
                    "token reuse detected; please sign in again");
            }
            case REVOKED, EXPIRED -> throw new BusinessException(ResponseCode.INVALID_TOKEN);
            case ACTIVE -> {}
        }
        if (!current.isUsable(now)) throw new BusinessException(ResponseCode.INVALID_TOKEN);

        var newOne = issue(current.userId(), device, ip);
        current.rotate(newOne.id(), now);
        tokens.save(current);
        return newOne;
    }

    @Transactional
    public void revoke(String rawIncoming) {
        var hash = sha256Hex(rawIncoming);
        tokens.findByTokenHash(hash).ifPresent(rt -> {
            rt.revoke();
            tokens.save(rt);
        });
    }
}

public record IssuedRefreshToken(String raw, RefreshTokenId id, Instant expiresAt, UserId userId) {}
```

> **핵심**:
> - raw 토큰은 클라에만, **DB 에는 SHA-256 hash**
> - ROTATED 상태 토큰이 다시 사용되면 = 도난 → 모든 RT REVOKE
> - 자세히: [[token-refresh-impl]]

---

## 7. Rate Limit — 로그인 5회 실패 lock

```java
@Component
@RequiredArgsConstructor
public class LoginAttemptLimiter {

    private final RedisTemplate<String, String> redis;
    private static final int MAX_FAILURES = 5;
    private static final Duration LOCK = Duration.ofMinutes(15);

    public void checkLimit(String ip, String email) {
        var key = "login:fail:" + ip + ":" + email;
        var count = redis.opsForValue().get(key);
        if (count != null && Integer.parseInt(count) >= MAX_FAILURES) {
            throw new BusinessException(ResponseCode.FORBIDDEN,
                "로그인 시도 횟수 초과. " + LOCK.toMinutes() + "분 후 다시 시도해 주세요.");
        }
    }

    public void recordFailure(String ip, String email) {
        var key = "login:fail:" + ip + ":" + email;
        var count = redis.opsForValue().increment(key);
        if (count == 1) redis.expire(key, LOCK);
    }

    public void recordSuccess(String ip, String email) {
        redis.delete("login:fail:" + ip + ":" + email);
    }
}
```

---

## 8. Controller

```java
@Tag(name = "사용자 로그인")
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class LoginController {

    private final LoginUseCase loginUseCase;
    private final RefreshTokenService rtService;

    @Operation(summary = "로그인")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "(OK_001) 로그인 성공"),
        @ApiResponse(responseCode = "401", description = "(UNAUTH_001) invalid email or password"),
        @ApiResponse(responseCode = "403", description = "(FORBID_001) 활성 계정이 아님")
    })
    @SecurityRequirements()
    @PostMapping("/login")
    public ResponseEntity<CommonResponse<TokenResponse>> login(
        @Valid @RequestBody LoginRequest req,
        HttpServletRequest http
    ) {
        var device = http.getHeader("User-Agent");
        var ip = ClientIpUtil.resolveClientIp(http);
        var result = loginUseCase.login(req.email(), req.password(), device, ip);
        var body = CommonResponse.success(ResponseCode.OK,
            new TokenResponse(result.accessToken(), result.refreshToken(), "Bearer", 900L),
            "로그인 성공"
        );
        return ResponseEntity.ok(body);
    }

    @Operation(summary = "로그아웃 — refresh token 무효화")
    @SecurityRequirements()
    @PostMapping("/logout")
    public ResponseEntity<CommonResponse<Void>> logout(@Valid @RequestBody RefreshRequest req) {
        rtService.revoke(req.refreshToken());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "로그아웃"));
    }
}

public record LoginRequest(
    @Email @NotBlank String email,
    @NotBlank @Size(min = 8, max = 128) String password
) {
    @Override public String toString() {
        return "LoginRequest[email=%s, password=***]".formatted(email);
    }
}

public record RefreshRequest(@NotBlank String refreshToken) {
    @Override public String toString() { return "RefreshRequest[refreshToken=***]"; }
}

public record TokenResponse(
    String accessToken, String refreshToken, String tokenType, long expiresIn
) {}
```

---

## 9. JpaRefreshTokenRepositoryAdapter

```java
// infrastructure/persistence/jpa/auth/RefreshTokenJpaEntity.java
@Entity
@Table(name = "refresh_tokens")
public class RefreshTokenJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "user_id", nullable = false, length = 26) private String userId;
    @Column(name = "token_hash", nullable = false, length = 64) private String tokenHash;
    @Column(name = "device_fingerprint", length = 255) private String deviceFingerprint;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private RefreshToken.Status status;
    @Column(name = "rotated_to_id", length = 26) private String rotatedToId;
    @Column(name = "issued_at", nullable = false) private Instant issuedAt;
    @Column(name = "expires_at", nullable = false) private Instant expiresAt;
    // 생성자 / 매핑 메서드
}

public interface RefreshTokenJpaRepository extends JpaRepository<RefreshTokenJpaEntity, String> {
    Optional<RefreshTokenJpaEntity> findByTokenHash(String tokenHash);

    @Modifying(clearAutomatically = true)
    @Query("update RefreshTokenJpaEntity rt set rt.status = 'REVOKED' " +
           "where rt.userId = :userId and rt.status = 'ACTIVE'")
    int revokeAllForUser(@Param("userId") String userId);
}

@Repository
@RequiredArgsConstructor
public class JpaRefreshTokenRepositoryAdapter implements RefreshTokenRepository {
    private final RefreshTokenJpaRepository spring;
    private final Clock clock;
    // save / findByTokenHash / revokeAllForUser 구현 + 도메인 ↔ JPA 매핑
}
```

DB 스키마는 [[database]] 의 `refresh_tokens` 테이블.

---

## 10. SecurityConfig — 인증 필요한 endpoint 보호

→ [[../../common/security-config]] 의 표준 SecurityConfig + 다음 추가:

```java
.requestMatchers(
    "/api/v1/auth/login",
    "/api/v1/auth/login/**",
    "/api/v1/auth/token/refresh",
    "/api/v1/auth/logout"
).permitAll()
.anyRequest().authenticated()
```

JwtAuthenticationFilter 가 access token 검증 + SecurityContext 주입.

---

## 11. 함정 모음

### 함정 1 — refresh 를 DB 에 raw 저장
DB 유출 = 모든 RT 사용 가능. **SHA-256 hash 저장**.

### 함정 2 — refresh rotation 없음
탈취된 RT 영구 사용. rotation + reuse detection.

### 함정 3 — 로그인 실패 메시지가 enumeration
"존재하지 않는 이메일" vs "비밀번호 틀림" 분리 = 가입자 목록 노출. **항상 "invalid email or password"**.

### 함정 4 — timing 차이로 enumeration
user 가 없으면 비교 X → 응답 빠름. user 있으면 hash 비교 → 느림. 차이로 가입자 enumeration. **dummy hash 비교** 강제.

### 함정 5 — access token 을 localStorage 에
XSS 한 번에 탈취. **httpOnly 쿠키** 권장 (단 CORS / CSRF 정책 같이).

### 함정 6 — 5회 실패 제한 없음
brute force 가능. Redis 카운터 + 15분 lock.

### 함정 7 — JWT 에 민감정보 (password / 주민번호 등)
JWT 는 디코드만 하면 다 보임. **claim 에 PII X**.

### 함정 8 — logout 후 access 도 무효 기대
JWT stateless — 만료까지 살아있음. 짧은 TTL 로 감수 또는 access 블랙리스트 (Redis).

### 함정 9 — `Algorithm: none` 수용
jjwt 0.12+ 는 기본 차단. 옛 라이브러리 명시 거부.

### 함정 10 — 시계 차이 (clock skew)
서버 간 1초 차이로 만료 검증 어긋남. **NTP 동기화 + `clockSkewSeconds(30)`**.

---

## 12. 운영 체크리스트

- [ ] `JWT_SECRET` 256-bit base64 + vault 주입
- [ ] access TTL 짧게 (15분)
- [ ] refresh DB `token_hash` UNIQUE
- [ ] 만료 RT cleanup 스케줄 (매일)
- [ ] login endpoint rate limit (5/min per IP+email)
- [ ] 로그인 실패 메트릭 + spike 알람
- [ ] RT reuse detection 발생 시 사용자 이메일 / 푸시 알림
- [ ] `JwtAuthenticationEntryPoint` 가 401 응답 (job-answer-be 의 400 버그 주의 — [[../../common/security-config#4.1]])

---

## 13. 관련

- [[signup|↑ hub]]
- [[phone-verification-impl]] — 이전 (§11)
- [[token-refresh-impl]] — 다음 (§13)
- [[../../common/security-config]] — JwtTokenProvider / JwtAuthenticationFilter
- [[../oauth2-social-login]] — 소셜 로그인 분기
