---
title: "소셜 로그인 — Apple / Google / Kakao / Naver"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - oauth2
  - social-login
---

# 소셜 로그인 — Apple / Google / Kakao / Naver

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 4종 provider / token verify / signupForSocial 흐름 |

**[[api-design|↑ api-design hub]]**

> 📐 **ORM**: §6 는 JPA Adapter sketch. 공통: [[../common/response-envelope]] · [[../common/security-config]].
>
> 전제: [[signup]] 의 `User` entity 가 `providerType` (LOCAL / APPLE / GOOGLE / KAKAO / NAVER) 와 `appleSub` 컬럼 보유.

---

## 1. 무엇을 만드는가

```
POST /api/v1/auth/login/social        # 소셜 토큰 → 우리 access/refresh 발급
POST /api/v1/auth/signup/social       # 소셜 토큰 + 약관 동의 → 신규 가입
GET  /api/v1/auth/login/social/redirect/{provider}    # OAuth code → 토큰 교환 (웹 흐름)
```

### 1.1 클라이언트 흐름 (2가지)

**A — 앱 (Kakao SDK / Google Sign-In SDK)**:
```
1. 앱에서 provider SDK 로 사용자 인증
2. SDK 가 access_token (또는 id_token) 반환
3. 앱이 우리 서버에 `POST /auth/login/social` { provider, accessToken } 전달
4. 서버: 토큰 검증 → 우리 사용자 lookup → 우리 JWT 발급
```

**B — 웹 (OAuth code 흐름)**:
```
1. 클라가 provider 의 authorize URL 로 redirect
2. provider 가 사용자 동의 후 redirect_uri 로 code 전달
3. 우리 서버가 code → token 교환 (server-side, 시크릿 사용)
4. 검증된 사용자 정보로 JWT 발급
```

본 레시피는 **A (앱 식)** 중심. B 는 §8.

### 1.2 비기능

- provider 별 토큰 검증 통일된 port (`SocialAuthProvider`)
- email 미제공 (Apple) 케이스 명시적 처리 — `SOCIAL_EMAIL_MISSING` 422
- 우리 DB 에 동일 email 의 다른 provider 가입 있으면 — 정책 결정 (병합 / 거절)
- 신규 가입은 별도 endpoint (`signup/social`) — 약관 동의 강제

---

## 2. 도메인 / Enum

```java
// src/main/java/com/example/shop/domain/user/SocialProviderType.java
public enum SocialProviderType {
    LOCAL, APPLE, GOOGLE, KAKAO, NAVER
}

// 도메인 record — provider 가 반환한 사용자 정보 (정규화 후)
public record SocialUserInfo(
    SocialProviderType provider,
    String externalId,         // provider 의 user ID (sub / openid)
    String email,              // null 가능 (Apple 일부 케이스)
    String name,
    String profileImageUrl,
    String appleSub            // Apple 만 (refresh 검증용)
) {}
```

### 2.1 User entity 확장

```java
// signup 레시피의 User 에 추가:
// - providerType (LOCAL / APPLE / GOOGLE / KAKAO / NAVER)
// - externalId (provider 의 user ID — provider 별 unique)
// - appleSub (Apple 의 refresh 검증)
// - password 는 LOCAL 만 (소셜은 null)
```

DB 마이그레이션:
```sql
ALTER TABLE users
  ADD COLUMN provider_type VARCHAR(20) NOT NULL DEFAULT 'LOCAL',
  ADD COLUMN external_id  VARCHAR(100),
  ADD COLUMN apple_sub    VARCHAR(100),
  ALTER COLUMN password_hash DROP NOT NULL;     -- 소셜은 null
CREATE UNIQUE INDEX ux_users_provider_external
  ON users (provider_type, external_id) WHERE external_id IS NOT NULL;
```

---

## 3. SocialAuthProvider — 통일 port

```java
// domain/auth/SocialAuthProvider.java
public interface SocialAuthProvider {
    SocialProviderType supports();
    SocialUserInfo verify(String accessToken);          // raw 토큰 → 검증된 user info
}
```

### 3.1 Google 구현

```java
// infrastructure/external/oauth/GoogleSocialAuthProvider.java
@Component
@RequiredArgsConstructor
public class GoogleSocialAuthProvider implements SocialAuthProvider {

    private final RestClient http;     // Bean — default baseUrl = https://oauth2.googleapis.com

    @Override public SocialProviderType supports() { return SocialProviderType.GOOGLE; }

    @Override
    public SocialUserInfo verify(String accessToken) {
        try {
            // tokeninfo endpoint — id_token 검증
            var info = http.get()
                .uri(uri -> uri.path("/tokeninfo").queryParam("id_token", accessToken).build())
                .retrieve()
                .body(GoogleTokenInfo.class);
            if (info == null || info.email() == null) {
                throw new BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 Google 토큰");
            }
            return new SocialUserInfo(
                SocialProviderType.GOOGLE,
                info.sub(),
                info.email(),
                info.name(),
                info.picture(),
                null
            );
        } catch (HttpStatusCodeException e) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED,
                "Google 인증 실패: " + e.getStatusCode());
        }
    }

    public record GoogleTokenInfo(
        String sub, String email, String name, String picture,
        @JsonProperty("email_verified") Boolean emailVerified
    ) {}
}
```

### 3.2 Kakao 구현

```java
@Component
@RequiredArgsConstructor
public class KakaoSocialAuthProvider implements SocialAuthProvider {

    private final RestClient http;       // baseUrl = https://kapi.kakao.com

    @Override public SocialProviderType supports() { return SocialProviderType.KAKAO; }

    @Override
    public SocialUserInfo verify(String accessToken) {
        try {
            var info = http.get()
                .uri("/v2/user/me")
                .header("Authorization", "Bearer " + accessToken)
                .retrieve()
                .body(KakaoUserInfo.class);

            if (info == null) {
                throw new BusinessException(ResponseCode.UNAUTHORIZED, "Kakao 응답이 비어 있음");
            }
            var email = info.kakaoAccount() == null ? null : info.kakaoAccount().email();
            var nickname = info.kakaoAccount() == null || info.kakaoAccount().profile() == null
                ? null : info.kakaoAccount().profile().nickname();
            var image = info.kakaoAccount() == null || info.kakaoAccount().profile() == null
                ? null : info.kakaoAccount().profile().profileImageUrl();
            return new SocialUserInfo(
                SocialProviderType.KAKAO,
                String.valueOf(info.id()),
                email,
                nickname,
                image,
                null
            );
        } catch (HttpStatusCodeException e) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED,
                "Kakao 인증 실패: " + e.getStatusCode());
        }
    }

    public record KakaoUserInfo(Long id, @JsonProperty("kakao_account") KakaoAccount kakaoAccount) {}
    public record KakaoAccount(String email, KakaoProfile profile) {}
    public record KakaoProfile(String nickname, @JsonProperty("profile_image_url") String profileImageUrl) {}
}
```

### 3.3 Apple 구현 (JWT 서명 검증)

Apple 은 access token 이 아니라 **id_token (JWT)** 사용. JWKS 로 서명 검증.

```java
@Component
public class AppleSocialAuthProvider implements SocialAuthProvider {

    private final JwksProvider jwks;          // https://appleid.apple.com/auth/keys 캐시

    @Override public SocialProviderType supports() { return SocialProviderType.APPLE; }

    @Override
    public SocialUserInfo verify(String idToken) {
        try {
            // 1. JWT 의 header.kid 추출 → JWKS 에서 매칭 key
            var signedJwt = SignedJWT.parse(idToken);
            var kid = signedJwt.getHeader().getKeyID();
            var key = jwks.getKey(kid)
                .orElseThrow(() -> new BusinessException(ResponseCode.UNAUTHORIZED,
                    "Apple key not found"));

            // 2. 서명 검증
            var verifier = new RSASSAVerifier(key);
            if (!signedJwt.verify(verifier)) {
                throw new BusinessException(ResponseCode.UNAUTHORIZED, "Apple 서명 불일치");
            }

            // 3. claims 검증 (issuer / audience / exp)
            var claims = signedJwt.getJWTClaimsSet();
            if (!"https://appleid.apple.com".equals(claims.getIssuer())) {
                throw new BusinessException(ResponseCode.UNAUTHORIZED, "Apple issuer 불일치");
            }
            if (claims.getExpirationTime().before(new Date())) {
                throw new BusinessException(ResponseCode.UNAUTHORIZED, "Apple 토큰 만료");
            }

            var email = claims.getStringClaim("email");           // 첫 로그인 후 null 가능
            var sub = claims.getSubject();

            return new SocialUserInfo(
                SocialProviderType.APPLE,
                sub,
                email,                                              // null 가능
                null,                                               // Apple 은 name 별도 (첫 로그인 시만)
                null,
                sub                                                 // appleSub = sub
            );
        } catch (ParseException | JOSEException e) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED,
                "Apple 토큰 검증 실패: " + e.getMessage());
        }
    }
}
```

> **Apple 함정**: 첫 로그인 시에만 email/name 제공. 이후 호출엔 sub 만. **sub 기반 lookup** 필수.

### 3.4 Naver 구현 (간단)

```java
// kakao 와 유사 — https://openapi.naver.com/v1/nid/me + Bearer header
// 응답의 response.id / response.email / response.name / response.profile_image
```

### 3.5 Provider Registry

```java
@Component
public class SocialAuthProviderRegistry {

    private final Map<SocialProviderType, SocialAuthProvider> providers;

    public SocialAuthProviderRegistry(List<SocialAuthProvider> all) {
        this.providers = all.stream()
            .collect(Collectors.toMap(SocialAuthProvider::supports, p -> p));
    }

    public SocialAuthProvider of(SocialProviderType type) {
        var p = providers.get(type);
        if (p == null) throw new BusinessException(ResponseCode.BAD_REQUEST,
            "지원하지 않는 소셜 로그인 타입: " + type);
        return p;
    }
}
```

→ Spring 이 자동으로 List<SocialAuthProvider> 로 모든 구현 주입. Strategy 패턴.

---

## 4. UseCase

### 4.1 SocialLoginUseCase

```java
// src/main/java/com/example/shop/application/auth/SocialLoginUseCase.java
@Service
@RequiredArgsConstructor
@Slf4j
public class SocialLoginUseCase {

    private final SocialAuthProviderRegistry providers;
    private final UserRepository users;
    private final JwtIssuer jwt;
    private final RefreshTokenService rtService;
    private final Clock clock;

    @Transactional
    public LoginResult login(SocialProviderType provider, String externalToken,
                             String device, String ip) {
        // 1. provider 검증
        SocialUserInfo info = providers.of(provider).verify(externalToken);

        // 2. 우리 DB lookup — provider + externalId 우선, fallback email
        var user = users.findByProviderAndExternalId(provider, info.externalId())
            .or(() -> info.email() == null
                ? Optional.empty()
                : users.findByEmail(new Email(info.email())))
            .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND,
                "가입된 계정이 없습니다. 회원가입을 진행해 주세요."));

        // 3. providerType 일관성 (이미 LOCAL 가입했는데 소셜 로그인 시도)
        if (user.providerType() != provider) {
            throw new BusinessException(ResponseCode.CONFLICT,
                "이미 다른 방식 (" + user.providerType() + ") 으로 가입된 이메일입니다.");
        }

        if (!user.isActive()) {
            throw new BusinessException(ResponseCode.FORBIDDEN, "활성 계정이 아닙니다.");
        }

        // 4. 토큰 발급 ([[login-jwt]] 와 동일 흐름)
        var access = jwt.generateAccessToken(user.email().value(), user.id().value(),
                                             user.role().name(), generateSessionId());
        var rt = rtService.issue(user.id(), device, ip);
        return new LoginResult(access, rt.raw(), rt.expiresAt());
    }
}
```

### 4.2 SocialSignupUseCase

```java
@Service
@RequiredArgsConstructor
public class SocialSignupUseCase {

    private final SocialAuthProviderRegistry providers;
    private final UserRepository users;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public UserId signup(SocialProviderType provider, String externalToken,
                         List<TermsAgreement> termsAgreements, boolean marketingAgreed) {
        SocialUserInfo info = providers.of(provider).verify(externalToken);

        // email 없으면 422 (Apple 일부)
        if (info.email() == null || info.email().isBlank()) {
            throw new BusinessException(ResponseCode.SOCIAL_EMAIL_MISSING);
        }

        // 중복 확인
        users.findByEmail(new Email(info.email())).ifPresent(u -> {
            throw new BusinessException(ResponseCode.DUPLICATE_DATA,
                "이미 가입된 이메일입니다 (provider=" + u.providerType() + ").");
        });

        // 필수 약관 검증 (signup 과 동일)
        validateRequiredTerms(termsAgreements);

        var now = Instant.now(clock);
        var user = User.registerSocial(
            new UserId(ids.next()),
            new Email(info.email()),
            info.name() != null ? info.name() : info.email().split("@")[0],
            provider,
            info.externalId(),
            info.appleSub(),
            info.profileImageUrl(),
            now
        );
        var saved = users.save(user);

        // 약관 동의 저장 + 이벤트
        saved.pullDomainEvents().forEach(events::publishEvent);
        return saved.id();
    }
}
```

---

## 5. Controller

```java
@Tag(name = "소셜 로그인 / 가입")
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class SocialAuthController {

    private final SocialLoginUseCase loginUseCase;
    private final SocialSignupUseCase signupUseCase;

    @Operation(summary = "소셜 로그인")
    @SecurityRequirements()
    @PostMapping("/login/social")
    public ResponseEntity<CommonResponse<TokenResponse>> login(
        @Valid @RequestBody SocialLoginRequest req,
        HttpServletRequest http
    ) {
        var device = http.getHeader("X-DEVICE-UUID");
        var ip = ClientIpUtil.resolveClientIp(http);
        var result = loginUseCase.login(req.provider(), req.accessToken(), device, ip);
        var body = CommonResponse.success(ResponseCode.OK,
            new TokenResponse(result.accessToken(), result.refreshToken(), "Bearer", 900L),
            "소셜 로그인 성공");
        return ResponseEntity.ok(body);
    }

    @Operation(summary = "소셜 회원가입")
    @SecurityRequirements()
    @PostMapping("/signup/social")
    public ResponseEntity<CommonResponse<Map<String, String>>> signup(
        @Valid @RequestBody SocialSignupRequest req
    ) {
        var userId = signupUseCase.signup(
            req.provider(), req.accessToken(),
            req.termsAgreements(), req.marketingAgreed()
        );
        var body = CommonResponse.success(ResponseCode.OK,
            Map.of("userId", userId.value()), "소셜 회원가입 성공");
        return ResponseEntity.ok(body);
    }
}

public record SocialLoginRequest(
    @NotNull SocialProviderType provider,
    @NotBlank String accessToken
) {
    @Override public String toString() {
        return "SocialLoginRequest[provider=" + provider + ", accessToken=***]";
    }
}

public record SocialSignupRequest(
    @NotNull SocialProviderType provider,
    @NotBlank String accessToken,
    @Valid @NotNull List<TermsAgreement> termsAgreements,
    boolean marketingAgreed
) { /* toString 마스킹 */ }
```

---

## 6. 설정

```kotlin
// build.gradle.kts (추가)
implementation("com.nimbusds:nimbus-jose-jwt:9.40")     // Apple JWT 검증
```

```yaml
# application.yml
app:
  oauth:
    google:
      base-url: https://oauth2.googleapis.com
    kakao:
      base-url: https://kapi.kakao.com
    naver:
      base-url: https://openapi.naver.com
    apple:
      jwks-url: https://appleid.apple.com/auth/keys
      issuer: https://appleid.apple.com
      audience: ${APPLE_CLIENT_ID}             # Apple Developer 의 service ID
```

---

## 7. SecurityConfig

`/api/v1/auth/login/social` + `/api/v1/auth/signup/social` 비인증 (위에서 `permitAll`).

---

## 8. OAuth code 흐름 (웹) — 참고

```
1. GET /api/v1/auth/login/social/redirect/google
   → 302 redirect to https://accounts.google.com/o/oauth2/auth?...
2. 사용자 동의 → google 이 redirect_uri=https://shop/oauth/callback/google?code=ABC
3. GET /api/v1/auth/login/social/callback/google?code=ABC
   → 서버: code + client_secret → google /token endpoint → access_token
   → user info 조회 → 우리 JWT 발급 + redirect to frontend
```

Spring 의 `spring-boot-starter-oauth2-client` 가 이 흐름 자동화 — 본 레시피 범위 밖.

---

## 9. 함정 모음

### 함정 1 — Apple 의 email 부재
첫 로그인 후엔 sub 만 옴. **sub 기반 lookup + appleSub 컬럼 별도 저장**.

### 함정 2 — Google 의 access_token vs id_token
verify endpoint 는 id_token. access_token 으로 `/userinfo` 호출도 가능하지만 검증 부담.

### 함정 3 — Provider 응답 캐싱 안 함
같은 토큰으로 매번 외부 API 호출 = latency 부담. **짧은 TTL Redis 캐시** (1~5분).

### 함정 4 — provider 와 user 의 일관성
같은 email 이 LOCAL + GOOGLE 가입 둘 다 가능 — 정책 결정. 보통 **첫 가입 provider 만 허용**.

### 함정 5 — 토큰 검증 실패 시 generic 500
401 / 422 로 명확히. 클라가 재시도 / 재인증 분기.

### 함정 6 — Apple JWKS rate limit
공식 JWKS 가 자주 호출되면 Apple 이 차단. **Caffeine / Redis 캐시** (1시간 TTL).

### 함정 7 — externalId 가 String / Long 혼동
Kakao 는 Long (`id: 123456`), Google 은 String (`sub: "abc"`). **DB 는 String 으로 통일**.

### 함정 8 — 이메일 인증 자동 통과 가정
소셜 가입 = 이메일 검증됨 가정 OK (provider 가 검증). 단 status 는 직접 ACTIVE 로 — `email_verified` 컬럼은 true.

### 함정 9 — `email_verified` 가 false 인 Google 응답
드물게 `email_verified=false` — 거절 또는 별도 인증.

### 함정 10 — provider 별 응답 schema 변경
provider docs 변경에 따른 깨짐. **DTO 에 nullable 필드 + `@JsonIgnoreProperties(ignoreUnknown=true)`**.

---

## 10. 운영 체크리스트

- [ ] Apple JWKS 캐싱 (1h TTL)
- [ ] provider 응답 검증 (email_verified)
- [ ] provider type + externalId UNIQUE 인덱스
- [ ] 가입 시 email 미제공 (Apple) 처리 422
- [ ] CORS — provider redirect 도메인 허용 (필요 시)
- [ ] secret (client_secret, JWT audience) vault
- [ ] provider 별 API 변경 모니터링
- [ ] login 실패 (provider 토큰 무효) 의 timing 균일화 — brute force 차단

---

## 11. 관련

- [[signup]] — User entity 의 providerType 필드
- [[login-jwt]] — JWT 발급 흐름 공유
- [[email-verification]] — LOCAL 가입의 인증 (소셜은 자동)
- [[../common/security-config]] — JWT secret / CORS
- [[api-design|↑ api-design hub]]
