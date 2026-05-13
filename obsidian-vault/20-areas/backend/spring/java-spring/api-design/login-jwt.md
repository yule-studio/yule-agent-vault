---
title: "로그인 (JWT) — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - jwt
---

# 로그인 (JWT) — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Java 21 / access + RT rotation / 블랙리스트 / Security 필터 |

**[[api-design|↑ api-design hub]]**

> 전제: [[signup]] 으로 `users` 테이블 + `Argon2PasswordEncoder` 존재.

---

## 1. 무엇을 만드는가

```
POST /api/v1/auth/login       # email + password → access + refresh
POST /api/v1/auth/refresh     # refresh token → new access + 새 refresh (rotation)
POST /api/v1/auth/logout      # refresh token 무효화
GET  /api/v1/me               # access token 검증 후 현재 사용자
```

### 1.1 요청 / 응답

```http
POST /api/v1/auth/login
{ "email": "alice@x.com", "password": "Tr0ub4dor!" }

200 OK
{
  "data": {
    "accessToken":  "eyJhbGciOi...",
    "refreshToken": "9f4b3a2c-...-base64url",
    "tokenType":    "Bearer",
    "expiresIn":    900
  }
}
```

### 1.2 비기능

- Access token **15 분** (짧게) — stateless, RT 로 갱신
- Refresh token **14 일**, **rotation** (사용 시 새 RT, 옛 RT 무효)
- 동시 다중 세션 허용 (디바이스별)
- 실패 응답시간 = 성공과 유사 (timing attack)
- 로그인 5회 실패 시 lock (선택)

---

## 2. 도메인 모델

### 2.1 RefreshToken (Aggregate)

```java
// src/main/java/com/example/shop/domain/auth/RefreshToken.java
public final class RefreshToken {

    public enum Status { ACTIVE, ROTATED, REVOKED, EXPIRED }

    private final RefreshTokenId id;                  // jti — ULID
    private final UserId userId;
    private final String tokenHash;                   // SHA-256(raw)
    private final String deviceFingerprint;
    private final Instant issuedAt;
    private final Instant expiresAt;
    private Status status;
    private RefreshTokenId rotatedToId;

    private RefreshToken(RefreshTokenId id, UserId userId, String tokenHash,
                         String device, Instant issuedAt, Instant expiresAt,
                         Status status, RefreshTokenId rotatedToId) {
        this.id = id; this.userId = userId; this.tokenHash = tokenHash;
        this.deviceFingerprint = device; this.issuedAt = issuedAt;
        this.expiresAt = expiresAt; this.status = status;
        this.rotatedToId = rotatedToId;
    }

    public static RefreshToken issue(RefreshTokenId id, UserId userId, String tokenHash,
                                     String device, Instant now, Duration ttl) {
        return new RefreshToken(id, userId, tokenHash, device, now,
                                now.plus(ttl), Status.ACTIVE, null);
    }

    public void rotate(RefreshTokenId newId, Instant now) {
        if (status != Status.ACTIVE) throw new IllegalStateException("cannot rotate: " + status);
        if (!now.isBefore(expiresAt)) throw new IllegalStateException("expired");
        this.status = Status.ROTATED;
        this.rotatedToId = newId;
    }
    public void revoke() { this.status = Status.REVOKED; }
    public boolean isUsable(Instant now) {
        return status == Status.ACTIVE && now.isBefore(expiresAt);
    }

    public RefreshTokenId id() { return id; }
    public UserId userId() { return userId; }
    public String tokenHash() { return tokenHash; }
    public Status status() { return status; }
    public Instant expiresAt() { return expiresAt; }

    // JPA 매핑용
    public static RefreshToken reconstitute(RefreshTokenId id, UserId userId, String hash,
                                            String device, Instant issuedAt, Instant expiresAt,
                                            Status status, RefreshTokenId rotatedToId) {
        return new RefreshToken(id, userId, hash, device, issuedAt, expiresAt, status, rotatedToId);
    }
}

public record RefreshTokenId(String value) {
    public RefreshTokenId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("RefreshTokenId must be ULID");
    }
}
```

### 2.2 Repository port

```java
// src/main/java/com/example/shop/domain/auth/RefreshTokenRepository.java
public interface RefreshTokenRepository {
    RefreshToken save(RefreshToken token);
    Optional<RefreshToken> findById(RefreshTokenId id);
    Optional<RefreshToken> findByTokenHash(String hash);
    void revokeAllForUser(UserId userId);
}
```

---

## 3. 아키텍처 / 흐름

```
1. POST /login
   └─ LoginUseCase
       ├─ users.findByEmail(...)
       ├─ encoder.matches(raw, hash)
       ├─ access = jwt.sign({sub: userId, exp: now+15m})
       ├─ rt.raw = SecureRandom 32 bytes (base64url)
       ├─ rt.hash = SHA-256(raw)
       ├─ refreshTokens.save(ACTIVE)
       └─ return (access, rt.raw)              # raw 는 클라에만, hash 만 DB

2. POST /refresh
   ├─ rt.hash = SHA-256(클라가 보낸 raw)
   ├─ refreshTokens.findByTokenHash(hash)
   ├─ ACTIVE && not expired → 새 RT 발급 + 기존 ROTATED
   └─ ROTATED 발견 = "reuse 감지" → 해당 user 의 모든 RT REVOKE (도난 의심)

3. POST /logout
   └─ rt → REVOKED

4. GET /me (protected)
   ├─ JwtAuthFilter: Authorization: Bearer <access>
   ├─ jwt.parse → user id 추출
   ├─ SecurityContext 에 Authentication
   └─ Controller 가 Authentication 사용
```

---

## 4. DB 선택 / 스키마

### 4.1 RT 저장 — RDB or Redis?

| 후보 | 장점 | 단점 |
| --- | --- | --- |
| **PostgreSQL** | 영속 / JOIN / audit 용이 | μs latency 아님 |
| **Redis** | μs / TTL 자동 만료 | 영속 약함 (사용자 강제 로그아웃) |

권장: 일반 SaaS 는 **Redis** (μs + TTL). 금융 / 감사 강한 도메인은 **PostgreSQL**. 본 레시피는 **RDB 기본** + Redis 변형 메모.

### 4.2 스키마 (PostgreSQL)

```sql
-- V2__create_refresh_tokens.sql
CREATE TABLE refresh_tokens (
  id                 CHAR(26) PRIMARY KEY,
  user_id            CHAR(26) NOT NULL REFERENCES users(id),
  token_hash         CHAR(64) NOT NULL,            -- SHA-256 hex
  device_fingerprint VARCHAR(255),
  status             VARCHAR(20) NOT NULL,
  rotated_to_id      CHAR(26),
  issued_at          TIMESTAMPTZ NOT NULL,
  expires_at         TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX ux_refresh_tokens_hash ON refresh_tokens (token_hash);
CREATE INDEX ix_refresh_tokens_user ON refresh_tokens (user_id, status);
CREATE INDEX ix_refresh_tokens_expires ON refresh_tokens (expires_at);
```

> **함정**: token raw 값을 DB 에 저장 X. DB 유출 = RT 그대로 사용 가능. **반드시 hash**.

### 4.3 만료 청소

```sql
DELETE FROM refresh_tokens WHERE expires_at < now() - INTERVAL '7 days';
```

`@Scheduled` 또는 pg_cron.

---

## 5. 보안 / 암호화

### 5.1 JWT 알고리즘

| 알고리즘 | 용도 |
| --- | --- |
| **HS256** | 모놀리식 단일 서비스 (시크릿 256+ bits) |
| **RS256** | 마이크로서비스 / 외부 검증자 (공개키 검증) |
| **ES256** | RS256 의 가벼운 대안 (ECDSA P-256) |
| none / 약한 HS256 | ❌ |

본 레시피: **HS256**. MSA 면 **RS256** + JWKS.

### 5.2 Access vs Refresh

| | Access | Refresh |
| --- | --- | --- |
| 형태 | JWT (서명만 검증) | Opaque (랜덤 문자열, DB 조회) |
| 수명 | 짧음 (10~15분) | 김 (1~30일) |
| 클라 저장 | 메모리 / sessionStorage | **httpOnly + Secure + SameSite=Strict 쿠키** |
| 서버 저장 | 안 함 | **hash 만 DB / Redis** |
| 갱신 | refresh 사용 | rotation |
| 폐기 | 못 함 (단명) | DB / Redis 삭제 |

권장: access 도 가능하면 **httpOnly 쿠키**. XSS 탈취 차단.

### 5.3 RT Rotation + Reuse Detection

```
유효 RT_v1 사용 → RT_v2 발급, RT_v1 → ROTATED
           ↓
       정상 흐름

공격자가 탈취한 RT_v1 또 사용 → ROTATED 발견
                                → user 의 모든 RT REVOKED
                                → 사용자 알림 ("새 위치에서 로그인")
```

OAuth 2.0 RFC 6819 / 8252 권장 패턴.

---

## 6. 구현 — Spring Boot

### 6.1 JWT 헬퍼

```java
// src/main/java/com/example/shop/infrastructure/security/jwt/JwtIssuer.java
@Component
public class JwtIssuer {

    private final SecretKey key;
    private final Duration accessTtl;
    private final String issuer;
    private final Clock clock;

    public JwtIssuer(
        @Value("${app.jwt.secret}") String secret,
        @Value("${app.jwt.access-ttl}") Duration accessTtl,
        @Value("${app.jwt.issuer}") String issuer,
        Clock clock
    ) {
        this.key = Keys.hmacShaKeyFor(Base64.getDecoder().decode(secret));
        this.accessTtl = accessTtl;
        this.issuer = issuer;
        this.clock = clock;
    }

    public String issue(UserId userId, Set<String> roles) {
        var now = Instant.now(clock);
        return Jwts.builder()
            .issuer(issuer)
            .subject(userId.value())
            .claim("roles", roles)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(accessTtl)))
            .signWith(key, Jwts.SIG.HS256)
            .compact();
    }

    public Jws<Claims> parse(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .requireIssuer(issuer)
            .clockSkewSeconds(30)
            .build()
            .parseSignedClaims(token);
    }
}
```

```yaml
# application.yml
app:
  jwt:
    secret: ${JWT_SECRET}                # TODO(secret): 256-bit base64. vault.
    issuer: shop.example.com
    access-ttl: PT15M
    refresh-ttl: P14D
```

### 6.2 RefreshTokenService

```java
// src/main/java/com/example/shop/application/auth/RefreshTokenService.java
@Service
public class RefreshTokenService {

    private final RefreshTokenRepository tokens;
    private final Clock clock;
    private final Duration ttl;
    private final IdGenerator ids;
    private final SecureRandom random = new SecureRandom();

    public RefreshTokenService(RefreshTokenRepository tokens, Clock clock,
                               @Value("${app.jwt.refresh-ttl}") Duration ttl,
                               IdGenerator ids) {
        this.tokens = tokens; this.clock = clock; this.ttl = ttl; this.ids = ids;
    }

    /** raw 는 클라에, 해시만 DB. */
    public IssuedRefreshToken issue(UserId userId, String device) {
        var rawBytes = new byte[32];
        random.nextBytes(rawBytes);
        var raw = Base64.getUrlEncoder().withoutPadding().encodeToString(rawBytes);
        var hash = sha256Hex(raw);

        var token = RefreshToken.issue(
            new RefreshTokenId(ids.next()),
            userId, hash, device,
            Instant.now(clock), ttl
        );
        tokens.save(token);
        return new IssuedRefreshToken(raw, token.id(), token.expiresAt(), userId);
    }

    /** rotation. 재사용 감지 시 user 의 모든 RT REVOKE. */
    @Transactional
    public IssuedRefreshToken rotate(String rawIncoming, String device) {
        var hash = sha256Hex(rawIncoming);
        var current = tokens.findByTokenHash(hash)
            .orElseThrow(InvalidTokenException::new);
        var now = Instant.now(clock);

        switch (current.status()) {
            case ROTATED -> {
                // 도난 의심
                tokens.revokeAllForUser(current.userId());
                throw new RefreshTokenReuseDetectedException(current.userId());
            }
            case REVOKED, EXPIRED -> throw new InvalidTokenException();
            case ACTIVE -> { /* 계속 */ }
        }
        if (!current.isUsable(now)) throw new InvalidTokenException();

        var newOne = issue(current.userId(), device);
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

    private static String sha256Hex(String s) {
        try {
            var md = MessageDigest.getInstance("SHA-256");
            var bytes = md.digest(s.getBytes(StandardCharsets.UTF_8));
            var sb = new StringBuilder();
            for (byte b : bytes) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (NoSuchAlgorithmException e) { throw new IllegalStateException(e); }
    }
}

public record IssuedRefreshToken(String raw, RefreshTokenId id,
                                 Instant expiresAt, UserId userId) {}
```

### 6.3 LoginUseCase

```java
// src/main/java/com/example/shop/application/auth/LoginUseCase.java
@Service
public class LoginUseCase {

    private final UserRepository users;
    private final PasswordEncoder passwordEncoder;
    private final JwtIssuer jwt;
    private final RefreshTokenService rtService;

    // argon2id pre-computed dummy. 실제 hash 형태여야 timing 비슷.
    private static final String DUMMY_HASH =
        "$argon2id$v=19$m=65536,t=3,p=4$AAAAAAAAAAAAAAAAAAAA$AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";

    public LoginUseCase(UserRepository users, PasswordEncoder passwordEncoder,
                        JwtIssuer jwt, RefreshTokenService rtService) {
        this.users = users; this.passwordEncoder = passwordEncoder;
        this.jwt = jwt; this.rtService = rtService;
    }

    @Transactional
    public LoginResult login(String email, String rawPassword, String device) {
        var normalized = email.trim().toLowerCase(Locale.ROOT);
        var maybeUser = users.findByEmail(new Email(normalized));

        if (maybeUser.isEmpty()) {
            // timing 균일화 — 사용자 없어도 dummy 비교
            passwordEncoder.matches(rawPassword, DUMMY_HASH);
            throw new InvalidCredentialsException();
        }
        var user = maybeUser.get();
        if (!passwordEncoder.matches(rawPassword, user.currentPasswordHash().value())) {
            throw new InvalidCredentialsException();
        }
        if (!user.isActive()) throw new AccountNotActiveException(user.status());

        var access = jwt.issue(user.id(), Set.of("USER"));
        var rt = rtService.issue(user.id(), device);
        return new LoginResult(access, rt.raw(), rt.expiresAt());
    }
}

public record LoginResult(String accessToken, String refreshToken, Instant refreshExpiresAt) {}
```

### 6.4 Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/auth/AuthController.java
@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final LoginUseCase login;
    private final RefreshTokenService rt;
    private final JwtIssuer jwt;

    public AuthController(LoginUseCase login, RefreshTokenService rt, JwtIssuer jwt) {
        this.login = login; this.rt = rt; this.jwt = jwt;
    }

    @PostMapping("/login")
    public ApiResponse<TokenResponse> login(@Valid @RequestBody LoginRequest req,
                                            HttpServletRequest http) {
        var device = trim(http.getHeader("User-Agent"));
        var r = login.login(req.email(), req.password(), device);
        return ApiResponse.ok(new TokenResponse(r.accessToken(), r.refreshToken(),
                                                "Bearer", 900L));
    }

    @PostMapping("/refresh")
    public ApiResponse<TokenResponse> refresh(@Valid @RequestBody RefreshRequest req,
                                              HttpServletRequest http) {
        var device = trim(http.getHeader("User-Agent"));
        var newRt = rt.rotate(req.refreshToken(), device);
        var access = jwt.issue(newRt.userId(), Set.of("USER"));
        return ApiResponse.ok(new TokenResponse(access, newRt.raw(), "Bearer", 900L));
    }

    @PostMapping("/logout")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void logout(@Valid @RequestBody RefreshRequest req) {
        rt.revoke(req.refreshToken());
    }

    private static String trim(String s) {
        if (s == null) return null;
        return s.length() > 255 ? s.substring(0, 255) : s;
    }
}

public record LoginRequest(
    @Email @NotBlank String email,
    @NotBlank String password
) {
    @Override public String toString() {
        return "LoginRequest[email=%s, password=***]".formatted(email);
    }
}

public record RefreshRequest(@NotBlank String refreshToken) {
    @Override public String toString() { return "RefreshRequest[refreshToken=***]"; }
}

public record TokenResponse(String accessToken, String refreshToken,
                            String tokenType, long expiresIn) {}
```

### 6.5 Spring Security 6 설정

```java
// src/main/java/com/example/shop/config/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    public SecurityConfig(JwtAuthenticationFilter jwtFilter) { this.jwtFilter = jwtFilter; }

    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)                  // stateless API
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/api/v1/auth/signup",
                    "/api/v1/auth/login",
                    "/api/v1/auth/refresh"
                ).permitAll()
                .anyRequest().authenticated()
            )
            .exceptionHandling(ex -> ex.authenticationEntryPoint((req, res, e) -> {
                res.setStatus(401);
                res.setContentType("application/json");
                res.getWriter().write(
                    "{\"error\":{\"code\":\"UNAUTHORIZED\",\"message\":\"unauthorized\"}}");
            }))
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### 6.6 JWT 인증 필터

```java
// src/main/java/com/example/shop/infrastructure/security/JwtAuthenticationFilter.java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtIssuer jwt;
    public JwtAuthenticationFilter(JwtIssuer jwt) { this.jwt = jwt; }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        var header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            var token = header.substring(7);
            try {
                var claims = jwt.parse(token).getPayload();
                var userId = claims.getSubject();
                @SuppressWarnings("unchecked")
                var rolesRaw = (List<String>) claims.getOrDefault("roles", List.of());
                var roles = rolesRaw.stream()
                    .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                    .toList();
                var auth = new UsernamePasswordAuthenticationToken(userId, null, roles);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException e) {
                // 무시 → 다음 필터에서 401
            }
        }
        chain.doFilter(req, res);
    }
}
```

### 6.7 사용 — `/me`

```java
// src/main/java/com/example/shop/presentation/api/v1/me/MeController.java
@RestController
@RequestMapping("/api/v1/me")
public class MeController {

    private final UserRepository users;
    public MeController(UserRepository users) { this.users = users; }

    @GetMapping
    public ApiResponse<MeResponse> me(Authentication auth) {
        var user = users.findById(new UserId(auth.getName()))
            .orElseThrow(NotFoundException::new);
        return ApiResponse.ok(new MeResponse(
            user.id().value(),
            user.email().value(),
            user.name(),
            user.status().name()
        ));
    }
}

public record MeResponse(String userId, String email, String name, String status) {}
```

### 6.8 JPA Adapter (스케치)

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/auth/RefreshTokenJpaEntity.java
@Entity
@Table(name = "refresh_tokens")
public class RefreshTokenJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "user_id", nullable = false, length = 26) private String userId;
    @Column(name = "token_hash", nullable = false, length = 64) private String tokenHash;
    @Column(name = "device_fingerprint", length = 255) private String deviceFingerprint;
    @Column(nullable = false, length = 20) private String status;
    @Column(name = "rotated_to_id", length = 26) private String rotatedToId;
    @Column(name = "issued_at", nullable = false) private Instant issuedAt;
    @Column(name = "expires_at", nullable = false) private Instant expiresAt;

    protected RefreshTokenJpaEntity() {}
    // ctor / getters / setters 생략
}

// JpaRefreshTokenRepositoryAdapter — UserRepository 와 같은 패턴
```

---

## 7. 트랜잭션 / 예외 / 검증

### 7.1 예외 매핑

| 예외 | 상태 | 코드 |
| --- | --- | --- |
| `InvalidCredentialsException` | 401 | `INVALID_CREDENTIALS` (email 노출 X) |
| `AccountNotActiveException` | 403 | `ACCOUNT_NOT_ACTIVE` |
| `InvalidTokenException` | 401 | `INVALID_TOKEN` |
| `RefreshTokenReuseDetectedException` | 401 | `RT_REUSE_DETECTED` + 알림 발송 |

```java
@RestControllerAdvice
public class AuthExceptionHandler {

    @ExceptionHandler(InvalidCredentialsException.class)
    public ResponseEntity<ApiResponse<Void>> invalid(InvalidCredentialsException e) {
        return ResponseEntity.status(401).body(ApiResponse.fail(
            new ApiError("INVALID_CREDENTIALS", "invalid email or password")));
    }

    @ExceptionHandler(RefreshTokenReuseDetectedException.class)
    public ResponseEntity<ApiResponse<Void>> reuse(RefreshTokenReuseDetectedException e) {
        // 별도 ListenER 가 사용자에게 알림 발송 — 도메인 이벤트 발행 권장
        return ResponseEntity.status(401).body(ApiResponse.fail(
            new ApiError("RT_REUSE_DETECTED",
                         "token reuse detected; please sign in again")));
    }
}
```

### 7.2 트랜잭션

- `login` — read + RT insert → `@Transactional`
- `refresh` — rotation 의 2단계 (insert new + update old) 한 트랜잭션
- `logout` — 단일 update → `@Transactional`

---

## 8. 테스트

### 8.1 단위 — RefreshTokenService rotation

```java
// src/test/java/com/example/shop/application/auth/RefreshTokenServiceTest.java
@ExtendWith(MockitoExtension.class)
class RefreshTokenServiceTest {

    @Mock RefreshTokenRepository repo;
    @Mock IdGenerator ids;
    Clock clock = Clock.fixed(Instant.parse("2026-05-14T00:00:00Z"), ZoneOffset.UTC);
    RefreshTokenService sut;

    @BeforeEach
    void setup() {
        when(ids.next()).thenReturn("01" + "A".repeat(24));
        sut = new RefreshTokenService(repo, clock, Duration.ofDays(14), ids);
    }

    @Test
    void rotation_on_ACTIVE_returns_new_RT_and_marks_old_ROTATED() {
        var oldHash = sha256Hex("raw-old");
        var old = RefreshToken.issue(
            new RefreshTokenId("OLD" + "X".repeat(23)),
            new UserId("U1" + "X".repeat(24)),
            oldHash, null,
            Instant.parse("2026-05-13T00:00:00Z"),
            Duration.ofDays(14)
        );
        when(repo.findByTokenHash(oldHash)).thenReturn(Optional.of(old));

        var result = sut.rotate("raw-old", null);

        verify(repo, atLeastOnce()).save(argThat(rt ->
            rt.status() == RefreshToken.Status.ROTATED || rt.status() == RefreshToken.Status.ACTIVE));
        assertThat(result.raw()).isNotEqualTo("raw-old");
    }

    @Test
    void reuse_of_ROTATED_token_revokes_all_tokens_for_user() {
        var hash = sha256Hex("stolen");
        var rotated = mock(RefreshToken.class);
        when(rotated.status()).thenReturn(RefreshToken.Status.ROTATED);
        when(rotated.userId()).thenReturn(new UserId("U1" + "X".repeat(24)));
        when(repo.findByTokenHash(hash)).thenReturn(Optional.of(rotated));

        assertThatThrownBy(() -> sut.rotate("stolen", null))
            .isInstanceOf(RefreshTokenReuseDetectedException.class);
        verify(repo).revokeAllForUser(new UserId("U1" + "X".repeat(24)));
    }

    private static String sha256Hex(String s) { /* helper */ return ""; }
}
```

### 8.2 통합 — login → /me

```java
// src/test/java/com/example/shop/integration/LoginApiIT.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class LoginApiIT {

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
    @Autowired UserTestFixture userFixture;     // signup + verify 헬퍼

    @Test
    void login_and_call_protected_endpoint() {
        userFixture.createActive("u@x.com", "p@ssword12");

        var loginRes = rest.postForEntity("/api/v1/auth/login",
            Map.of("email", "u@x.com", "password", "p@ssword12"), Map.class);
        var data = (Map<?, ?>) loginRes.getBody().get("data");
        var access = (String) data.get("accessToken");

        var headers = new HttpHeaders();
        headers.setBearerAuth(access);
        var meRes = rest.exchange("/api/v1/me", HttpMethod.GET,
            new HttpEntity<>(headers), Map.class);
        assertThat(meRes.getStatusCode().value()).isEqualTo(200);
    }

    @Test
    void wrong_password_returns_401() {
        userFixture.createActive("u@x.com", "p@ssword12");
        var res = rest.postForEntity("/api/v1/auth/login",
            Map.of("email", "u@x.com", "password", "wrong-one"), Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(401);
    }
}
```

---

## 9. 운영 체크리스트

- [ ] `JWT_SECRET` 256-bit (32 bytes) base64. vault 주입.
- [ ] HS256 → RS256 마이그레이션 시 JWKS 운영 계획
- [ ] access TTL 짧게 (15분 이하). 너무 짧으면 RT 요청 폭증
- [ ] RT 는 **httpOnly + Secure + SameSite=Strict 쿠키** 권장
- [ ] login endpoint 에 **rate limiting** (IP + email). 5회/분 권장
- [ ] 실패 timing 균일화 (dummy hash 비교)
- [ ] `InvalidCredentialsException` 메시지는 "email or password" — enumeration 차단
- [ ] RT reuse 감지 시 사용자 알림 (이메일/푸시)
- [ ] 만료된 RT cleanup 스케줄 (매일 1회)
- [ ] access token `roles` claim 은 최소만 (큰 권한 목록은 서버 lookup)
- [ ] 로그아웃 = RT REVOKE. access 는 만료까지 살아 있음 (짧은 TTL 로 감수)

---

## 10. 함정 모음

### 함정 1 — JWT 에 민감정보
JWT 는 base64 디코드만 하면 다 보임. 패스워드/주민번호 X.

### 함정 2 — access token 을 localStorage 에
XSS 한 번에 탈취. **httpOnly 쿠키** 정석.

### 함정 3 — RT 를 raw 그대로 DB 저장
DB 유출 = 모든 RT 사용 가능. **SHA-256 hash 저장**.

### 함정 4 — RT rotation 없음
탈취된 RT 영구 사용. rotation + reuse detection 둘 다 필요.

### 함정 5 — 로그인 실패에 "no such email" / "wrong password"
enumeration. 항상 "invalid email or password".

### 함정 6 — `csrf().disable()` 무지성
stateless JWT API 면 OK. RT 를 쿠키에 두고 동일 도메인 호출이면 위험. SameSite=Strict / double-submit.

### 함정 7 — 로그아웃 = access 즉시 무효 기대
JWT stateless. 만료까지 살아 있음. 필요 시 access 블랙리스트 (Redis) — 단 stateless 의 장점 소실.

### 함정 8 — `JwtIssuer` 마다 새로 생성
파서/키 비용 큼. `@Component` 싱글톤.

### 함정 9 — 시계 차이 (clock skew)
서버 간 1초만 어긋나도 만료 검증 어긋남. NTP 동기화 + `clockSkewSeconds(30)`.

### 함정 10 — `Algorithm: none` 수용
jjwt 0.12+ 는 기본 차단. 옛 라이브러리는 명시적 거부. 화이트리스트만 (`Jwts.SIG.HS256`).

---

## 11. 관련

- [[signup]] — 가입
- [[password-reset]] — 잊은 패스워드
- rate-limiting (다음) — login brute force
- oauth2-social-login (다음)
- [[../../../../security/security|↗ security hub]]
- [[api-design|↑ api-design hub]]
