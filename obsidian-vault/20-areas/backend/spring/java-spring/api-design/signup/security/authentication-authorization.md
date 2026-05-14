---
title: "인증 / 인가 — 누가 호출하나"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - authn
  - authz
---

# 인증 / 인가 — 누가 호출하나

**[[security|↑ security hub]]**

> Spring Security 의 filter chain 으로 endpoint 별 권한 정의. 잘못 설정 시 **민감 endpoint 가 비인증 노출** 또는 **정상 사용자가 reject** 폭증.

---

## 1. 인증 (Authentication) vs 인가 (Authorization)

| 구분 | 무엇 |
| --- | --- |
| 인증 (authn) | "누구인지" — JWT 검증 |
| 인가 (authz) | "무엇을 할 수 있는지" — role 검증 |

---

## 2. Endpoint 권한 매트릭스

### 2.1 anonymous (비인증)

| Endpoint | 왜 anonymous |
| --- | --- |
| `POST /auth/signup` | 가입 자체가 비인증 |
| `POST /auth/login` | 로그인 자체가 비인증 |
| `POST /auth/verify/email/confirm` | 토큰만 검증 (user 식별 안 됨) |
| `POST /auth/verify/phone/request` | 가입 전 휴대폰 인증 |
| `POST /auth/password-reset/request` | 비밀번호 잊은 사용자 |
| `POST /auth/password-reset/confirm` | 토큰만 검증 |
| `POST /auth/refresh` | refresh token 검증 |
| `GET /docs/**` | Swagger 등 |
| `GET /health` / `/actuator/health` | 헬스체크 |

### 2.2 authenticated (인증)

| Endpoint | 무엇 |
| --- | --- |
| `GET /me` | 본인 정보 |
| `PATCH /me` | 본인 정보 변경 |
| `POST /me/password` | 비밀번호 변경 (+ step-up) |
| `GET /me/sessions` | 디바이스 목록 |
| `DELETE /me/sessions/{id}` | 디바이스 logout |
| `POST /me/2fa/setup` | 2FA 설정 |
| `POST /auth/logout` | refresh revoke |

### 2.3 ADMIN

| Endpoint | 무엇 |
| --- | --- |
| `POST /admin/users` | admin 이 user 추가 |
| `PATCH /admin/users/{id}/status` | user 정지 / 해제 |
| `DELETE /admin/users/{id}` | user 강제 삭제 |
| `GET /admin/users?...` | user 검색 |
| `GET /admin/audit-logs` | audit log 조회 |

### 2.4 MASTER (최상위)

| Endpoint | 무엇 |
| --- | --- |
| `POST /admin/admins` | ADMIN 권한 부여 / 해제 |
| `GET /admin/system-stats` | 시스템 통계 |

---

## 3. Spring SecurityConfig

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)             // JWT — CSRF 불필요
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // 비인증 (anonymous)
                .requestMatchers(HttpMethod.POST,
                    "/api/v1/auth/signup",
                    "/api/v1/auth/login",
                    "/api/v1/auth/refresh",
                    "/api/v1/auth/verify/email/confirm",
                    "/api/v1/auth/verify/phone/**",
                    "/api/v1/auth/password-reset/**"
                ).permitAll()
                .requestMatchers(
                    "/docs/**", "/swagger-ui/**", "/v3/api-docs/**",
                    "/actuator/health", "/actuator/info"
                ).permitAll()

                // ADMIN
                .requestMatchers("/api/v1/admin/admins/**").hasRole("MASTER")
                .requestMatchers("/api/v1/admin/**").hasAnyRole("ADMIN", "MASTER")

                // 나머지 — authenticated
                .anyRequest().authenticated()
            )
            .exceptionHandling(eh -> eh
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

자세히: [[../../common/security-config]].

---

## 4. 왜 STATELESS

- JWT 기반 — 서버가 session 안 저장.
- 매 요청마다 JWT 검증.
- Spring Session 의 Redis 같은 외부 의존 없음.

**왜 SessionCreationPolicy.STATELESS**
- IF_REQUIRED (default) → request 마다 session 생성 → 메모리 부담.
- STATELESS → session 안 만듦.

---

## 5. 왜 CSRF disabled

- CSRF 는 cookie 기반 인증에 필요.
- JWT 가 Authorization header → cross-origin 자동 차단 (browser 가 header 안 보냄).
- CSRF token 추가 부담 X.

**언제 enabled**
- refresh token 이 HttpOnly cookie 시 → CSRF token 같이 사용.
- 본 vault: refresh 도 cookie → CSRF token 활성화 옵션.

---

## 6. CORS 화이트리스트

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(
        "https://app.example.com",
        "https://staging.example.com"
    ));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Request-Id"));
    config.setAllowCredentials(true);                   // cookie refresh
    config.setMaxAge(3600L);

    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

**왜 명시적 origin (`*` 아님)**
- `*` + allowCredentials=true = CORS 사양 위반.
- `*` 는 모든 origin 허용 = 보안 risk.

**안 하면 무슨 문제**
- 다른 도메인의 악의적 site 에서 사용자 JWT 도용.

---

## 7. JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider tokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        try {
            var token = resolveToken(req);
            if (token != null) {
                var auth = tokenProvider.parseAndAuthenticate(token);
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        } catch (JwtException e) {
            SecurityContextHolder.clearContext();
            // 401 응답은 EntryPoint 가 처리
        }
        chain.doFilter(req, res);
    }

    private String resolveToken(HttpServletRequest req) {
        var bearer = req.getHeader("Authorization");
        return bearer != null && bearer.startsWith("Bearer ")
            ? bearer.substring(7)
            : null;
    }
}
```

**왜 try/catch + clearContext (throw 안 함)**
- JWT 검증 실패해도 filter chain 계속 — anonymous 로 진행.
- authorizeHttpRequests 가 anonymous = denied 시 자동 401.

---

## 8. 권한 검증 (`@PreAuthorize`)

```java
@PreAuthorize("hasRole('ADMIN')")
@PatchMapping("/admin/users/{id}/status")
public void updateUserStatus(...) { ... }

@PreAuthorize("hasRole('ADMIN') and #userId == authentication.principal.id")
@GetMapping("/admin/users/{userId}")
public UserResponse getUser(@PathVariable String userId, Authentication authentication) { ... }
```

**왜 `@PreAuthorize` (SecurityConfig 외)**
- 메서드 단 권한 — 동적 표현 가능 (`#userId == principal.id`).
- 코드 가까이 = 권한 변경 시 한 곳만 수정.

**활성화**
```java
@Configuration
@EnableMethodSecurity     // @PreAuthorize 활성화
public class SecurityConfig { ... }
```

---

## 9. Authentication Principal

```java
@GetMapping("/me")
public UserResponse me(@AuthenticationPrincipal AuthUser authUser) {
    return userService.findById(authUser.id());
}
```

```java
public record AuthUser(UserId id, Role role) implements UserDetails {
    @Override public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    // 나머지 UserDetails 메서드
}
```

**왜 AuthUser (UserDetails 가 아님)**
- UserDetails 의 username / password 필드는 무의미 (JWT 기반).
- AuthUser 가 도메인 의미 (UserId / Role) 만 노출.

---

## 10. 함정 모음

### 함정 1 — `permitAll()` 너무 광범위
실수로 `/admin/**` permitAll → 비인증 ADMIN 접근.
→ 명시적 path matching + 정기 audit.

### 함정 2 — STATELESS 안 함
session 메모리 부담 + auto-scaling 시 sticky session 필요.
→ STATELESS 명시.

### 함정 3 — CORS `*` + credentials
사양 위반 + 보안 사고.
→ 화이트리스트.

### 함정 4 — JWT 검증 실패 시 500
filter 에서 throw → 500 응답 (의도는 401).
→ try/catch + clearContext.

### 함정 5 — `@PreAuthorize` 활성화 안 함 (`@EnableMethodSecurity` 누락)
annotation 무시 → 권한 우회.
→ 명시.

### 함정 6 — `hasRole('ADMIN')` 의 ROLE prefix 헷갈림
Spring 이 자동 `ROLE_` prefix. DB 에 `ADMIN` 저장 후 `hasRole('ADMIN')` 매핑.
→ enum 그대로 ('ADMIN') 사용.

### 함정 7 — ADMIN 강등 시 즉시 반영 안 됨
JWT 안의 role 이 15분 유효 → 강등 후 15분 동안 ADMIN.
→ 강등 시 강제 logout (RT revokeAll) + access 짧게.

### 함정 8 — anonymous 인데 JWT 보내면 error
일부 endpoint (signup) 가 anonymous + 로그인된 user 가 호출 시 → 받음 vs 거절 정책.
→ 받음 (단순).

### 함정 9 — `/actuator/**` 모두 노출
민감 정보 (env, configprops, heapdump) 노출.
→ health / info 만.

### 함정 10 — `/h2-console` 같은 dev 도구 prod 노출
실수로 활성.
→ dev profile 만.

### 함정 11 — CSRF disabled 인데 cookie 사용
cookie + CSRF 없으면 cross-site 요청 통과.
→ CSRF enabled + token 같이.

### 함정 12 — UserDetails 의 password 필드 noop
실제 검증은 JWT 가 함. password 필드 비워두면 헷갈림.
→ 별도 AuthUser 정의.

---

## 11. 관련

- [[security|↑ security hub]]
- [[../../common/security-config]] — SecurityConfig 상세
- [[../design-decisions/token-model]] — JWT 정책
- [[../login-impl]] — 로그인 흐름
- [[../enums/role]] — 권한 매트릭스
