---
title: "SecurityConfig + JWT Filter + CORS — 표준 세팅"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:30:00+09:00
tags:
  - backend
  - java-spring
  - common
  - security
  - jwt
  - cors
  - production
---

# SecurityConfig + JWT Filter + CORS — 표준 세팅

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | job-answer-be 추출 + 버그 수정 + 보강 |

**[[../java-spring|↑ Java Spring]]**

> 본 문서가 **Spring Security 6 표준 세팅의 본거지**. 모든 레시피의 `SecurityConfig` 는 여기 패턴.
> [[response-envelope]] 의 `CommonResponse / ResponseCode` 를 401/403 응답에 사용.

---

## 1. 컴포넌트 한눈에

```
HttpSecurity (Spring Security 6 람다 DSL)
   │
   ├─ cors                       ← CorsConfigurationSource (yaml 의 cors 속성)
   ├─ csrf(disable)              ← stateless API
   ├─ httpBasic(disable)
   ├─ sessionManagement: STATELESS
   ├─ authorizeHttpRequests
   │     ├ public:    /api/auth/**, /swagger-ui/**, /actuator/health, ...
   │     ├ MASTER:    /admin/key/**
   │     ├ ADMIN:     /admin/**
   │     └ 그 외:     authenticated()
   ├─ exceptionHandling
   │     └ authenticationEntryPoint = JwtAuthenticationEntryPoint  (401 응답)
   ├─ addFilterBefore: ApiLogFilter ─ JwtAuthenticationFilter ─ UsernamePasswordAuthenticationFilter
   │   (요청 들어오는 순서: ApiLog → Jwt → Username)
   └─ build()

빈:
  PasswordEncoder = BCryptPasswordEncoder
  CorsConfigurationSource (CorsProperties 주입)
```

---

## 2. `SecurityConfig`

```java
// src/main/java/com/example/shop/common/config/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)        // @PreAuthorize 활성
@RequiredArgsConstructor
@EnableConfigurationProperties(CorsProperties.class)
public class SecurityConfig {

    private final CorsProperties corsProperties;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final ApiLogFilter apiLogFilter;
    private final JwtAccessDeniedHandler jwtAccessDeniedHandler;  // 403 — 신규 추가 (원본엔 없음)

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .cors(c -> c.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .formLogin(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)    // 401 — 미인증
                .accessDeniedHandler(jwtAccessDeniedHandler)              // 403 — 인가 실패
            )
            .authorizeHttpRequests(auth -> auth
                // ----- 정적 / 공개 -----
                .requestMatchers(
                    "/.well-known/**",
                    "/swagger-ui.html", "/swagger-ui/**",
                    "/v3/api-docs/**", "/swagger-resources/**", "/webjars/**",
                    "/actuator/health", "/actuator/info", "/actuator/prometheus",
                    "/favicon.ico"
                ).permitAll()
                // ----- 인증 / 사용자 / 약관 -----
                .requestMatchers(
                    "/api/v1/auth/signup", "/api/v1/auth/signup/**",
                    "/api/v1/auth/login",  "/api/v1/auth/login/**",
                    "/api/v1/auth/token/refresh",
                    "/api/v1/auth/password/**",
                    "/api/v1/auth/duplicate/**",
                    "/api/v1/auth/verify/**"          // 전화번호 / 이메일 인증
                ).permitAll()
                // ----- 메시지 / 외부 webhook -----
                .requestMatchers(
                    "/api/v1/messages/webhooks/**",
                    "/api/v1/payments/webhooks/**"
                ).permitAll()
                // ----- WebSocket -----
                .requestMatchers("/api/v1/ws/**").permitAll()
                // ----- 관리자 -----
                .requestMatchers("/admin/key/**").hasRole("MASTER")
                .requestMatchers("/admin/**").hasAnyRole("ADMIN", "MASTER")
                // ----- 그 외 -----
                .anyRequest().authenticated()
            )
            // 필터 순서: 가장 먼저 들어오는 요청부터 — ApiLog → Jwt → (UsernamePassword)
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(apiLogFilter, JwtAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        var config = new CorsConfiguration();
        config.setAllowCredentials(false);                          // JWT + Authorization header → credentials false 권장
        config.setAllowedOrigins(splitOrEmpty(corsProperties.getAllowedOrigins()));
        config.setAllowedMethods(splitOrEmpty(corsProperties.getAllowedMethods()));
        config.setAllowedHeaders(splitOrEmpty(corsProperties.getAllowedHeaders()));
        if (corsProperties.getMaxAge() != null) {
            config.setMaxAge(corsProperties.getMaxAge());
        }
        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    private static List<String> splitOrEmpty(String csv) {
        if (csv == null || csv.isBlank()) return List.of();
        return Arrays.stream(csv.split(",")).map(String::trim).filter(s -> !s.isEmpty()).toList();
    }
}
```

### 2.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `accessDeniedHandler` 없음 (403 시 default) | `JwtAccessDeniedHandler` 추가 → 일관된 envelope |
| `formLogin` 비활성 명시 안 함 | `.formLogin(disable)` 추가 |
| `corsProperties.getAllowedOrigins().split(",")` 직접 | null / blank 방어 + trim |
| URL list 가 길고 흩어짐 | 그룹별 주석 정렬 |
| `/api/*/user/login/**` 같은 wildcard | path versioning (`/api/v1/auth/login`) 으로 정리 |

---

## 3. `JwtAuthenticationFilter`

```java
// src/main/java/com/example/shop/common/config/jwt/JwtAuthenticationFilter.java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserQueryService userQueryService;        // emailHash + soft-delete 인지

    private static final List<String> SWAGGER_PATHS = List.of(
        "/swagger-ui/**", "/swagger-ui.html",
        "/v3/api-docs/**", "/swagger-resources/**", "/webjars/**"
    );
    private static final PathMatcher PATH_MATCHER = new AntPathMatcher();

    @Override
    protected boolean shouldNotFilter(HttpServletRequest req) {
        // Swagger / 정적 리소스는 JWT 검증 스킵 (성능 + 디버깅 편의)
        return SWAGGER_PATHS.stream().anyMatch(p -> PATH_MATCHER.match(p, req.getServletPath()));
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {

        String token = jwtTokenProvider.resolveToken(req);

        if (StringUtils.hasText(token)
            && jwtTokenProvider.isAccess(token)
            && jwtTokenProvider.validateToken(token)) {

            String email = jwtTokenProvider.getEmail(token);
            String sid = jwtTokenProvider.getSessionId(token);          // 멀티 세션 / 강제 로그아웃 용

            // emailHash + softDelete 필터로 조회 — 평문 unique 가능성 회피
            User user = userQueryService.findActiveByEmail(email);

            if (user != null && Objects.equals(email, user.getEmail())) {
                // (선택) 서버 측 세션 무효화 검증 — 강제 로그아웃 시
                if (sid != null && !userQueryService.isSessionActive(user.getId(), sid)) {
                    log.warn("invalidated session for user={}", user.getId());
                } else {
                    var authorities = List.<GrantedAuthority>of(
                        new SimpleGrantedAuthority(user.getRole().name())
                    );
                    var auth = new UsernamePasswordAuthenticationToken(user, null, authorities);
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        }
        chain.doFilter(req, res);
    }
}
```

### 3.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `UserRepository` 직접 import (도메인 격리 X) | `UserQueryService` 통과 (도메인 / 인프라 격리) |
| `sid` claim 발급 후 검증 안 함 | 서버 측 세션 활성 검증 추가 |
| `logger.debug("...")` 평문 | `@Slf4j` + 파라미터화 |
| `findByEmailHashAndDeletedAtIsNull` 노출 (인프라 누수) | service 메서드명 (`findActiveByEmail`) |
| Swagger path 가 코드에 hard-coded | `static final List` 분리 + AntPathMatcher 1회 생성 |

> **핵심**: 토큰 자체만 검증 → DB 까지 가서 사용자 살아있는지 / 권한 / 세션 확인. JWT stateless 의 한계를 보강.

---

## 4. `JwtAuthenticationEntryPoint` — 401 응답

```java
// src/main/java/com/example/shop/common/config/jwt/JwtAuthenticationEntryPoint.java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;     // Bean 주입 (전역 Jackson 설정 활용)

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        var body = CommonResponse.<Void>fail(
            ResponseCode.UNAUTHORIZED,
            "인증이 필요합니다."
        );

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);          // 401 (원본 버그 수정)
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());
        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

### 4.1 원본의 버그 (수정)

> ⚠️ **버그**: 원본 `job-answer-be` 의 `JwtAuthenticationEntryPoint` 는 `response.setStatus(HttpServletResponse.SC_BAD_REQUEST)` (400) — **401 응답해야 할 자리에 400 응답**. 클라이언트가 토큰 만료 / 미발급 케이스를 인증 실패로 못 잡음. **반드시 SC_UNAUTHORIZED (401)** 로 수정.

다른 개선:
- `ObjectMapper` 매번 새로 생성 → `@Bean` 주입 (한 번)
- `setCharacterEncoding(UTF-8)` 추가 (한글 메시지 깨짐 방지)
- `MediaType.APPLICATION_JSON_VALUE` 상수 사용

---

## 5. `JwtAccessDeniedHandler` — 403 응답 (신규)

```java
// src/main/java/com/example/shop/common/config/jwt/JwtAccessDeniedHandler.java
@Component
@RequiredArgsConstructor
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException ex) throws IOException {
        var body = CommonResponse.<Void>fail(
            ResponseCode.INSUFFICIENT_PERMISSION,
            "해당 리소스에 접근할 권한이 없습니다."
        );
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());
        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

→ 원본엔 없는 컴포넌트. `@PreAuthorize` 실패 / role 부족 시 일관된 envelope 응답.

---

## 6. `CorsProperties` — yaml 주입

```java
// src/main/java/com/example/shop/common/properties/CorsProperties.java
@ConfigurationProperties(prefix = "cors")
@Getter @Setter
public class CorsProperties {
    private String allowedOrigins;     // "https://shop.example.com,https://admin.example.com"
    private String allowedMethods;     // "GET,POST,PUT,PATCH,DELETE,OPTIONS"
    private String allowedHeaders;     // "*" 또는 명시
    private Long maxAge;                // 3600 (s)
}
```

```yaml
# application.yml
cors:
  allowedOrigins: ${CORS_ORIGINS:http://localhost:3000}
  allowedMethods: GET,POST,PUT,PATCH,DELETE,OPTIONS
  allowedHeaders: Authorization,Content-Type,Accept-Language,X-DEVICE-UUID
  maxAge: 3600
```

```yaml
# application-prod.yml
cors:
  allowedOrigins: https://shop.example.com,https://admin.example.com
```

> **함정**: `*` (모든 origin 허용) + `setAllowCredentials(true)` 조합은 브라우저가 거부. JWT + Authorization 헤더는 `credentials = false` 권장 ([[../api-docs/swagger-springdoc]] 의 brower CORS 절 참고).

---

## 7. JwtTokenProvider — 표준 (jjwt 0.12)

```java
// src/main/java/com/example/shop/common/config/jwt/JwtTokenProvider.java
@Slf4j
@Component
public class JwtTokenProvider {

    private static final String CLAIM_ID    = "id";        // 사용자 PK
    private static final String CLAIM_ROLE  = "role";
    private static final String CLAIM_TYPE  = "token_type"; // ACCESS / REFRESH
    private static final String CLAIM_JTI   = "jti";
    private static final String CLAIM_SID   = "sid";        // 세션 UUID (멀티 세션 / 강제 로그아웃)

    public static final String AUTHORIZATION_HEADER = "Authorization";
    public static final String BEARER_PREFIX = "Bearer ";

    private final SecretKey key;
    private final Duration accessTtl;
    private final Duration refreshTtl;
    private final String issuer;

    public JwtTokenProvider(
        @Value("${jwt.secret}") String secret,
        @Value("${jwt.access-minutes}") long accessMinutes,
        @Value("${jwt.refresh-days}") long refreshDays,
        @Value("${jwt.issuer:shop.example.com}") String issuer
    ) {
        // ⚠️ secret 은 Base64 권장. 원본은 raw bytes 사용. 256-bit 이상 필수.
        var keyBytes = Base64.getDecoder().decode(secret);
        if (keyBytes.length < 32) {
            throw new IllegalStateException("JWT secret 은 256-bit (32 bytes) 이상이어야 합니다.");
        }
        this.key = Keys.hmacShaKeyFor(keyBytes);
        this.accessTtl = Duration.ofMinutes(accessMinutes);
        this.refreshTtl = Duration.ofDays(refreshDays);
        this.issuer = issuer;
    }

    public String generateAccessToken(String email, Long userId, String role, String sessionId) {
        return build(email, userId, normalizeRole(role), "ACCESS", accessTtl, sessionId);
    }

    public String generateRefreshToken() {
        return build(null, null, null, "REFRESH", refreshTtl, null);
    }

    private String build(String email, Long userId, String role, String type,
                         Duration ttl, String sessionId) {
        var now = Instant.now();
        var builder = Jwts.builder()
            .issuer(issuer)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(ttl)))
            .claim(CLAIM_TYPE, type)
            .id(UUID.randomUUID().toString())            // jti
            .signWith(key, Jwts.SIG.HS256);

        if ("ACCESS".equals(type)) {
            if (email != null) builder.subject(email);
            if (userId != null) builder.claim(CLAIM_ID, userId);
            if (role != null) builder.claim(CLAIM_ROLE, role);
            if (sessionId != null) builder.claim(CLAIM_SID, sessionId);
        }
        // REFRESH 는 jti + iss + iat + exp 만. PII / role 없음.
        return builder.compact();
    }

    public Claims parseClaims(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .requireIssuer(issuer)
            .clockSkewSeconds(30)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public boolean validateToken(String token) {
        if (!StringUtils.hasText(token)) return false;
        try { parseClaims(token); return true; }
        catch (ExpiredJwtException e) { log.debug("expired"); }
        catch (JwtException e) { log.warn("invalid JWT: {}", e.getMessage()); }
        catch (IllegalArgumentException e) { log.warn("empty JWT claims"); }
        return false;
    }

    public boolean isAccess(String token) {
        try { return "ACCESS".equals(parseClaims(token).get(CLAIM_TYPE, String.class)); }
        catch (JwtException e) { return false; }
    }

    public boolean isRefresh(String token) {
        try { return "REFRESH".equals(parseClaims(token).get(CLAIM_TYPE, String.class)); }
        catch (JwtException e) { return false; }
    }

    public String getEmail(String token)     { return parseClaims(token).getSubject(); }
    public String getJti(String token)        { return parseClaims(token).getId(); }
    public Long   getUserId(String token)     { return parseClaims(token).get(CLAIM_ID, Long.class); }
    public String getRole(String token)       { return normalizeRole(parseClaims(token).get(CLAIM_ROLE, String.class)); }
    public String getSessionId(String token)  { return parseClaims(token).get(CLAIM_SID, String.class); }
    public Instant getExpiration(String token){ return parseClaims(token).getExpiration().toInstant(); }

    public String resolveToken(HttpServletRequest req) {
        var bearer = req.getHeader(AUTHORIZATION_HEADER);
        if (StringUtils.hasText(bearer) && bearer.startsWith(BEARER_PREFIX)) {
            return bearer.substring(BEARER_PREFIX.length());
        }
        return null;
    }

    private static String normalizeRole(String raw) {
        if (raw == null || raw.isBlank()) return "ROLE_USER";
        return raw.startsWith("ROLE_") ? raw : "ROLE_" + raw;
    }
}
```

### 7.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `secret.getBytes()` raw | `Base64.getDecoder().decode(secret)` 권장 + 길이 검증 |
| issuer 검증 안 함 | `requireIssuer` + parse 시 검증 |
| clock skew 없음 | `clockSkewSeconds(30)` |
| `setIssuedAt`, `setExpiration` (옛 API) | 0.12 의 `issuedAt`, `expiration` |
| jti 를 `claim(CLAIM_JTI, ...)` 으로 직접 | jjwt 의 `id(...)` (표준 JWT id claim) |
| `getUserIdOrNull` 의 NumberFormatException 처리 | `Long.class` 로 직접 파싱 |
| `ExpiredJwtException` 핸들링 없음 | 별도 catch (운영 로그 노이즈 줄임) |

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET}                            # base64 인코드된 256-bit+ 시크릿. vault.
  access-minutes: 15                               # 짧게
  refresh-days: 14
  issuer: shop.example.com
```

---

## 8. ApiLogFilter — 모든 요청 추적

[[response-envelope#9]] 의 ApiLogFilter 패턴.

```java
// src/main/java/com/example/shop/common/filter/ApiLogFilter.java
@Component
@RequiredArgsConstructor
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class ApiLogFilter extends OncePerRequestFilter {

    private final ApiCallLogRepository logs;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(req, res);
        } finally {
            Exception ex = (Exception) req.getAttribute(ApiExceptionHandler.EXCEPTION_ATTR);
            logs.recordAsync(new ApiCallLog(
                req.getMethod(),
                req.getRequestURI(),
                queryString(req),
                res.getStatus(),
                System.currentTimeMillis() - start,
                clientIp(req),
                req.getHeader("X-DEVICE-UUID"),
                userIdFromContext(),
                ex == null ? null : ex.getClass().getSimpleName(),
                ex == null ? null : truncate(ex.getMessage(), 500)
            ));
        }
    }

    private static String clientIp(HttpServletRequest req) {
        var xff = req.getHeader("X-Forwarded-For");
        return (xff != null && !xff.isBlank()) ? xff.split(",")[0].trim() : req.getRemoteAddr();
    }
    // 기타 helper
}
```

운영 dashboard 에서:
- 5xx 비율 / endpoint 별 latency p95
- 같은 IP / device 의 짧은 시간 다회 요청 (brute force 의심)
- 특정 user 의 모든 action 추적

---

## 9. yaml 표준 세팅 (전체)

```yaml
# application.yml
spring:
  jackson:
    time-zone: UTC                             # 직렬화 시 UTC 일관
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false        # ISO-8601 문자열
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate

jwt:
  secret: ${JWT_SECRET}
  access-minutes: 15
  refresh-days: 14
  issuer: shop.example.com

cors:
  allowedOrigins: ${CORS_ORIGINS:http://localhost:3000}
  allowedMethods: GET,POST,PUT,PATCH,DELETE,OPTIONS
  allowedHeaders: Authorization,Content-Type,Accept-Language,X-DEVICE-UUID
  maxAge: 3600

logging:
  level:
    org.springframework.security: INFO
    com.example.shop: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: never                      # 운영 노출 X
```

---

## 10. 운영 체크리스트

- [ ] `JWT_SECRET` Base64 인코드 256-bit+. vault 주입.
- [ ] `accessMinutes ≤ 30` (보통 15)
- [ ] `refreshDays` 정책 명시 (보통 14~30)
- [ ] `corsProperties.allowedOrigins` 가 명시적 (와일드카드 X)
- [ ] `setAllowCredentials(false)` 또는 origin 와일드카드 X (둘 중 하나)
- [ ] `accessDeniedHandler` 존재 (403 일관 envelope)
- [ ] `authenticationEntryPoint` 가 **401 응답** (원본 버그 주의)
- [ ] 모든 public endpoint 명시 (whitelist), 그 외 `authenticated()`
- [ ] `actuator/prometheus` `actuator/health` 만 공개
- [ ] `BCryptPasswordEncoder` 또는 `argon2id` 사용 (강한 hash)
- [ ] CSRF disable + httpBasic disable + formLogin disable (stateless API 표준)
- [ ] 필터 순서 — ApiLog → Jwt → Username
- [ ] Swagger path 는 `shouldNotFilter` 에서 스킵 (성능)

---

## 11. 함정 모음 (원본에서 발견 / 수정)

### 함정 1 — `JwtAuthenticationEntryPoint` 가 400 응답 ⚠️
원본 `job-answer-be` 버그: 401 자리에 `SC_BAD_REQUEST` (400). 클라이언트가 토큰 만료 분기 못 함. **`SC_UNAUTHORIZED`** 로 수정.

### 함정 2 — `setAllowCredentials(true)` + `*` origin
브라우저가 거부. JWT + Authorization 헤더면 `false` 가 정상.

### 함정 3 — JWT secret 이 짧음
`secret.getBytes()` 의 length 가 256-bit 미만이면 HMAC-SHA256 가 약함. **반드시 32 bytes 이상** + Base64 인코드 권장.

### 함정 4 — issuer 검증 안 함
다른 시스템 / 옛 토큰이 우연히 같은 secret 으로 서명되면 통과. **`requireIssuer`**.

### 함정 5 — clock skew 없음
서버 간 시간 1초만 어긋나도 만료 검증 미스. `clockSkewSeconds(30)`.

### 함정 6 — refresh token 에 PII 포함
원본은 refresh 에 email 안 넣지만, 일부 프로젝트는 실수로 넣음. **opaque (jti only)** + 서버 매핑.

### 함정 7 — `accessDeniedHandler` 미설정
403 응답이 default — envelope 깨짐. 직접 핸들러 등록.

### 함정 8 — filter 순서
`ApiLogFilter` 가 `JwtAuthenticationFilter` 보다 먼저 와야 인증 실패 케이스도 로그.

### 함정 9 — `permitAll()` path 누락
새 endpoint 추가 후 SecurityConfig 갱신 안 함 → 401. 매 endpoint 의 인증 정책을 PR 체크리스트.

### 함정 10 — `@PreAuthorize` 만 믿고 Service 단에서 권한 미체크
직접 호출 (배치 / 다른 서비스) 시 우회 가능. **Service / Domain 단에도 체크**.

---

## 12. 관련

- [[response-envelope]] — 401/403/422 응답이 모두 이 envelope
- [[../api-docs/swagger-springdoc]] — Swagger 의 `@SecurityScheme(Bearer Authentication)`
- [[../api-design/login-jwt]] — 토큰 발급·갱신·로그아웃 흐름
- [[../pitfalls/concurrency-pitfalls]] (예정)
