---
title: "Swagger / springdoc-openapi — 본편"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:45:00+09:00
tags:
  - backend
  - java-spring
  - api-docs
  - swagger
  - springdoc
---

# Swagger / springdoc-openapi — 본편

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 설정 / 그룹 / 인증 / examples / yaml export / 운영 차단 |

**[[api-docs|↑ api-docs hub]]**

---

## 1. 의존성

```kotlin
// build.gradle.kts
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")     // Spring Boot 3.3
// WebFlux 사용 시:
// implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.6.0")
```

> Spring Boot 3 / Spring 6 / Java 17+. springfox 는 호환 X — 사용 X.

---

## 2. 기본 설정

```yaml
# application.yml
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true                              # 환경별로 끔 (§9)
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    operationsSorter: method                   # alphabetic / method / order
    tagsSorter: alpha
    tryItOutEnabled: true
    displayRequestDuration: true
    persistAuthorization: true                 # access token 한 번 입력 후 유지
    doc-expansion: none                        # none / list / full
  group-configs:
    - group: 'auth'
      paths-to-match: '/api/v1/auth/**'
      display-name: 'Auth'
    - group: 'commerce'
      paths-to-match: '/api/v1/products/**,/api/v1/cart/**,/api/v1/orders/**,/api/v1/payments/**'
      display-name: 'Commerce'
    - group: 'admin'
      paths-to-match: '/api/v1/admin/**'
      display-name: 'Admin (internal)'
  default-produces-media-type: application/json
  default-consumes-media-type: application/json
```

`/swagger-ui.html` 접속 → 그룹별 UI.

---

## 3. 전역 OpenAPI 정의

```java
// src/main/java/com/example/shop/config/OpenApiConfig.java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI(@Value("${app.name:Shop}") String appName,
                           @Value("${spring.profiles.active:local}") String profile) {
        return new OpenAPI()
            .info(new Info()
                .title(appName + " API")
                .version("v1")
                .description("Shop backend API. Profile: " + profile)
                .contact(new Contact().email("backend@example.com"))
                .license(new License().name("Internal").url("https://example.com")))
            .servers(List.of(
                new Server().url("https://api.example.com").description("prod"),
                new Server().url("https://staging-api.example.com").description("staging"),
                new Server().url("http://localhost:8080").description("local")
            ))
            // 인증 — §7
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                        .in(SecurityScheme.In.HEADER)
                        .name("Authorization"))
                .addSecuritySchemes("idempotencyKey",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.APIKEY)
                        .in(SecurityScheme.In.HEADER)
                        .name("Idempotency-Key")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}
```

---

## 4. Controller 어노테이션

### 4.1 기본

```java
@RestController
@RequestMapping("/api/v1/auth")
@Tag(name = "Auth", description = "회원가입 / 로그인 / 패스워드 리셋")
public class AuthController {

    @Operation(
        summary = "로그인 (Email + Password)",
        description = "성공 시 access token (JWT) + refresh token (opaque) 발급.",
        responses = {
            @ApiResponse(responseCode = "200", description = "성공"),
            @ApiResponse(responseCode = "401", description = "INVALID_CREDENTIALS",
                content = @Content(mediaType = "application/json",
                    examples = @ExampleObject("""
                        {"error":{"code":"INVALID_CREDENTIALS","message":"invalid email or password"}}
                        """))),
            @ApiResponse(responseCode = "403", description = "ACCOUNT_NOT_ACTIVE")
        }
    )
    @SecurityRequirements({})                                // 이 endpoint 는 비인증
    @PostMapping("/login")
    public ApiResponse<TokenResponse> login(@Valid @RequestBody LoginRequest req) {
        ...
    }
}
```

### 4.2 DTO `@Schema` — 필드 설명 / 예시

```java
@Schema(description = "회원가입 요청")
public record SignupRequest(
    @Schema(description = "이메일", example = "alice@example.com")
    @Email @NotBlank @Size(max = 254)
    String email,

    @Schema(description = "패스워드 (8~128자)", example = "Tr0ub4dor!", minLength = 8, maxLength = 128)
    @NotBlank @Size(min = 8, max = 128)
    String password,

    @Schema(description = "이름", example = "Alice")
    @NotBlank @Size(max = 100)
    String name,

    @Schema(description = "이용약관 동의 필수", example = "true")
    @AssertTrue boolean termsAgreed,

    @Schema(description = "마케팅 수신 동의 (선택)", example = "false")
    boolean marketingAgreed
) {}
```

→ Bean Validation 어노테이션이 자동으로 schema 제약으로 변환. `@Schema` 는 description / example 만 추가.

### 4.3 Parameter 보강

```java
@GetMapping
public ApiResponse<ProductListResponse> list(
    @Parameter(description = "검색어 (상품명 부분일치)") @RequestParam(required = false) String q,
    @Parameter(description = "브랜드 ID (다중 가능)") @RequestParam(required = false) List<String> brand,
    @Parameter(description = "정렬", schema = @Schema(implementation = SortKey.class))
        @RequestParam(required = false) SortKey sort,
    @Parameter(description = "페이지 한도 (1-100, 기본 20)",
               schema = @Schema(minimum = "1", maximum = "100", defaultValue = "20"))
        @RequestParam(defaultValue = "20") int limit
) { ... }
```

---

## 5. enum / sealed 표현

### 5.1 enum

```java
@Schema(description = "주문 상태", enumAsRef = true)
public enum OrderStatus {
    PENDING_PAYMENT, PAID, SHIPPED, DELIVERED, CANCELED, EXPIRED
}
```

→ Swagger 에서 별도 schema 로 떠서 참조됨.

### 5.2 sealed (oneOf)

```java
@Schema(oneOf = {MemberOwnerDto.class, GuestOwnerDto.class}, discriminatorProperty = "type")
public sealed interface CartOwnerDto permits MemberOwnerDto, GuestOwnerDto {}

@Schema(name = "MemberOwner")
public record MemberOwnerDto(String type, String userId) implements CartOwnerDto {}

@Schema(name = "GuestOwner")
public record GuestOwnerDto(String type, String guestId) implements CartOwnerDto {}
```

---

## 6. 응답 envelope / 표준화

```java
@Schema(description = "표준 응답 envelope")
public record ApiResponse<T>(
    @Schema(description = "성공 시 data, 실패 시 null")
    T data,
    @Schema(description = "실패 시 error, 성공 시 null")
    ApiError error
) {}

@Schema(description = "에러 응답")
public record ApiError(
    @Schema(example = "EMAIL_ALREADY_EXISTS") String code,
    @Schema(example = "이미 가입된 이메일") String message,
    @Schema(description = "필드별 상세 (validation 등)") Map<String, Object> details
) {}
```

### 6.1 표준 에러 응답을 모든 endpoint 에 자동 추가

```java
// 별도 OperationCustomizer 로 모든 operation 에 401/403/422/500 자동 추가
@Bean
public OperationCustomizer addStandardErrorResponses() {
    return (operation, handlerMethod) -> {
        var responses = operation.getResponses();
        addIfAbsent(responses, "401", "UNAUTHORIZED");
        addIfAbsent(responses, "403", "FORBIDDEN");
        addIfAbsent(responses, "422", "VALIDATION_FAILED");
        addIfAbsent(responses, "500", "INTERNAL_ERROR");
        return operation;
    };
}
```

→ 각 controller 에 일일이 안 적어도 표준 에러 응답이 문서에 나옴.

---

## 7. 인증 — Bearer / 헤더 표현

§3 에서 `bearerAuth` 정의. 사용:

```java
@SecurityRequirement(name = "bearerAuth")
@GetMapping("/me")
public ApiResponse<MeResponse> me(Authentication auth) { ... }

// 인증 불필요
@SecurityRequirements({})        // 비어 있으면 전역 requirement 무시
@PostMapping("/login")
public ApiResponse<TokenResponse> login(...) { ... }
```

Swagger UI 에서 우상단 `Authorize` 버튼 → JWT 입력 → 이후 `Try it out` 에서 헤더 자동 추가.

### 7.1 OAuth2 / OpenID

```java
.addSecuritySchemes("oauth2", new SecurityScheme()
    .type(SecurityScheme.Type.OAUTH2)
    .flows(new OAuthFlows()
        .authorizationCode(new OAuthFlow()
            .authorizationUrl("https://idp.example.com/oauth/authorize")
            .tokenUrl("https://idp.example.com/oauth/token")
            .scopes(new Scopes().addString("read", "read").addString("write", "write")))))
```

---

## 8. examples / 다중 예시

```java
@Operation(...)
@ApiResponses({
    @ApiResponse(responseCode = "201",
        content = @Content(
            mediaType = "application/json",
            examples = {
                @ExampleObject(name = "성공",
                    value = """
                        {"data":{"userId":"01HZ...","email":"alice@x.com","status":"PENDING_VERIFICATION"}}
                        """),
                @ExampleObject(name = "마케팅 동의 포함",
                    value = "...")
            }))
})
@PostMapping("/signup")
public ApiResponse<SignupResponse> signup(@Valid @RequestBody SignupRequest req) { ... }
```

Swagger UI 의 응답 영역에서 사용자가 예시 선택 가능.

### 8.1 RequestBody 의 examples

```java
@PostMapping("/products")
public ApiResponse<...> create(
    @RequestBody(
        required = true,
        content = @Content(
            schema = @Schema(implementation = CreateProductRequest.class),
            examples = {
                @ExampleObject(name = "기본",  value = """{"name":"후드","brandId":"..."}"""),
                @ExampleObject(name = "옵션 포함", value = """{"name":"후드","options":[...]}""")
            }
        )
    )
    @org.springframework.web.bind.annotation.RequestBody @Valid CreateProductRequest req
) { ... }
```

> **함정**: `io.swagger.v3.oas.annotations.parameters.RequestBody` 와 `org.springframework.web.bind.annotation.RequestBody` 가 같이 필요 — 이름 충돌 주의.

---

## 9. 환경별 노출 정책 — 운영 차단

### 9.1 profile 기반 활성

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false        # /v3/api-docs 비활성
  swagger-ui:
    enabled: false        # /swagger-ui.html 비활성
```

```yaml
# application-staging.yml
springdoc:
  api-docs:
    enabled: true
```

### 9.2 internal IP / VPN 만 허용 (Spring Security)

```java
@Configuration
public class SwaggerSecurityConfig {

    @Bean
    public SecurityFilterChain swaggerChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**")
                    .access(new WebExpressionAuthorizationManager(
                        "hasIpAddress('10.0.0.0/16') or hasIpAddress('127.0.0.1')"))
            )
            .httpBasic(Customizer.withDefaults())             // 추가로 basic auth
            .csrf(AbstractHttpConfigurer::disable)
            .build();
    }
}
```

### 9.3 basic auth

```yaml
spring:
  security:
    user:
      name: docs
      password: ${SWAGGER_DOC_PASSWORD}        # vault
```

---

## 10. OpenAPI yaml export / CI 통합

### 10.1 빌드 시 yaml 발행

```kotlin
// build.gradle.kts
plugins { id("org.springdoc.openapi-gradle-plugin") version "1.8.0" }

openApi {
    outputDir.set(file("$buildDir/docs"))
    outputFileName.set("openapi.yaml")
    apiDocsUrl.set("http://localhost:8080/v3/api-docs.yaml")
    waitTimeInSeconds.set(60)
    forkProperties.set("-Dspring.profiles.active=docs")
}

tasks.named("forkedSpringBootRun") {
    dependsOn("bootRun")
}
```

```bash
./gradlew generateOpenApiDocs
# build/docs/openapi.yaml 생성
```

### 10.2 CI 에서 검증 + 발행

```yaml
# .github/workflows/api-docs.yml
- name: Generate OpenAPI
  run: ./gradlew generateOpenApiDocs

- name: Lint OpenAPI (Spectral)
  run: npx @stoplight/spectral-cli lint build/docs/openapi.yaml --ruleset spectral.yaml

- name: Diff against main
  run: |
    git fetch origin main
    git show origin/main:build/docs/openapi.yaml > prev.yaml || true
    npx oasdiff diff prev.yaml build/docs/openapi.yaml --format markdown >> $GITHUB_STEP_SUMMARY

- name: Publish to docs site
  uses: actions/upload-artifact@v4
  with:
    name: openapi
    path: build/docs/openapi.yaml
```

### 10.3 breaking change 감지

`oasdiff` / `openapi-diff` — 이전 yaml 과 비교해서 breaking change 자동 감지. PR 에 코멘트.

---

## 11. 클라이언트 SDK 자동 생성 (옵션)

```bash
npx @openapitools/openapi-generator-cli generate \
  -i build/docs/openapi.yaml \
  -g typescript-axios \
  -o ../web/src/generated/api
```

→ 프론트가 직접 사용하는 TypeScript / Java / Python client 자동 생성.

---

## 12. REST Docs 와의 비교

| | springdoc-openapi | Spring REST Docs |
| --- | --- | --- |
| 생성 시점 | 런타임 (annotation 스캔) | 테스트 실행 (snippet) |
| 정확성 | annotation 잊으면 잘못된 문서 | 테스트 통과 = 문서 정확 |
| 비용 | 낮음 | 테스트 작성 필수 |
| 결과물 | OpenAPI 3 | asciidoc / 정적 site |
| 권장 | 일반 SaaS | 외부 공개 API / 엄격한 SLA |

→ 일반은 springdoc, 외부 API 는 둘 다.

---

## 13. 함정 모음

### 함정 1 — 운영 Swagger UI 가 그대로 열림
모든 endpoint / 인증 메커니즘 노출. **반드시 `enabled: false` 또는 IP 제한**.

### 함정 2 — Bean Validation 안 쓰면 schema 비어 있음
`@NotBlank`, `@Size` 등이 schema 제약으로 변환. **검증 어노테이션 충분히**.

### 함정 3 — `record` 의 컴팩트 constructor 가 문서에 안 보임
springdoc 은 필드 / accessor 만 본다. 검증 로직은 어차피 schema 에 안 들어감. **`@Schema(description=...)`** 으로 보강.

### 함정 4 — sealed interface 문서화 미흡
`oneOf` + `discriminator` 명시 안 하면 client SDK 가 못 만들어짐.

### 함정 5 — 둘의 `@RequestBody` 충돌
Spring 의 것은 메서드 파라미터, OpenAPI 의 것은 메서드 메타. 둘 다 명시.

### 함정 6 — 동일 endpoint 가 여러 그룹에 노출
`paths-to-match` 가 겹침. 사용자 혼동. 명확히 분리.

### 함정 7 — examples 의 JSON 오타
런타임 검증 X. 빌드 시 lint 또는 IT 로 확인.

### 함정 8 — `springfox` 사용
Spring Boot 3 호환 X. **`springdoc` 으로 마이그레이션**.

### 함정 9 — security scheme 정의 후 endpoint 에 안 적용
전역 SecurityRequirement 또는 endpoint 별 `@SecurityRequirement` 필요.

### 함정 10 — Swagger UI 의 CORS
같은 도메인이면 무관. 다른 도메인 (예: docs.example.com 에서 api.example.com 호출) 이면 CORS 설정 필요.

---

## 14. 운영 체크리스트

- [ ] **운영 환경에서 Swagger UI 와 `/v3/api-docs` 비활성** 또는 IP/basic auth 제한
- [ ] 모든 endpoint 에 `@Operation` summary / 표준 에러 응답
- [ ] DTO `record` 에 `@Schema` description / example
- [ ] 인증 endpoint 에 `@SecurityRequirement` 정확히
- [ ] CI 에서 yaml 생성 + lint + breaking change diff
- [ ] 그룹별 분리 (auth / commerce / admin)
- [ ] SDK 자동 생성 (필요 시)
- [ ] `Try it out` 의 prod URL 노출 여부 검토

---

## 15. 관련

- [[api-docs|↑ api-docs hub]]
- [[../api-design/api-design|↗ API 레시피]]
- [[../java-spring|↑ Java Spring]]
