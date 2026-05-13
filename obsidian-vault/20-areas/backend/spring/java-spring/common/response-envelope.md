---
title: "표준 응답 envelope + 글로벌 예외 처리"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:00:00+09:00
tags:
  - backend
  - java-spring
  - common
  - response
  - exception
  - production
---

# 표준 응답 envelope + 글로벌 예외 처리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | job-answer-be 실전 패턴 추출 + 개선 |

**[[../java-spring|↑ Java Spring]]**

> **이 문서가 본거지.** 모든 레시피 (`api-design/`) 는 여기 정의된 `CommonResponse` / `ResponseCode` / `BusinessException` / `ApiExceptionHandler` 를 그대로 사용.
> 실제 한국 SaaS production code 에서 추출한 패턴 + 명확히 개선.

---

## 1. 왜 envelope 통일이 첫 작업인가

- **클라이언트 (앱/웹) 가 한 가지 응답 모양만 처리** — 성공 / 실패 분기 단순.
- **에러 코드가 안정적** (`OK_001`, `BADREQ_002`, `UNAUTH_001` 같은 stable string) — 메시지 번역 / 분기 / 알람.
- **글로벌 예외 한 곳에 모음** — Controller 마다 try-catch 안 함.
- **운영 로그** 와 API 응답 코드가 1:1 — 장애 시 클라 측 코드만 보고 위치 추적.

---

## 2. 4 컴포넌트 한눈에

```
[Service] ── throw BusinessException(ResponseCode.X, "메시지") ──┐
                                                                 │
[Controller] ── return CommonResponse<T> ──────────────────┐     │
                                                            ▼     ▼
                                                ┌─────────────────┐
                                                │ ApiExceptionHandler  (@RestControllerAdvice)
                                                │   - BusinessException → CommonResponse(코드+메시지)
                                                │   - MethodArgumentNotValidException → 422
                                                │   - S3Exception / IOException / ... → 매핑
                                                │   - Exception (캐치올) → 500
                                                └─────────────────┘
                                                       │
                                                       ▼
                                            ResponseUtil.getHttpStatus(response)
                                                       │
                                                       ▼
                                        ResponseEntity<CommonResponse>
```

---

## 3. `CommonResponse<T>` — 응답 envelope

```java
// src/main/java/com/example/shop/common/model/dto/response/CommonResponse.java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CommonResponse<T> {

    private String code;            // "OK_001" / "BADREQ_002" — stable string
    private ResponseCode status;    // enum (HttpStatus 포함)
    private String message;         // 사람이 읽는 메시지 (i18n 가능)
    private T result;               // 성공 시 데이터. null 이면 직렬화 제외.

    // ---- factory methods ----

    public static <T> CommonResponse<T> success(ResponseCode code, String message) {
        return CommonResponse.<T>builder()
            .code(code.code())
            .status(code)
            .message(message)
            .build();
    }

    public static <T> CommonResponse<T> success(ResponseCode code, T data, String message) {
        return CommonResponse.<T>builder()
            .code(code.code())
            .status(code)
            .message(message)
            .result(data)
            .build();
    }

    public static <T> CommonResponse<T> fail(ResponseCode code) {
        return CommonResponse.<T>builder()
            .code(code.code())
            .status(code)
            .message(code.message())
            .build();
    }

    public static <T> CommonResponse<T> fail(ResponseCode code, String customMessage) {
        return CommonResponse.<T>builder()
            .code(code.code())
            .status(code)
            .message(customMessage != null ? customMessage : code.message())
            .build();
    }
}
```

> **개선 포인트 (원본 대비)**:
> - 제네릭 사용을 일관 (`CommonResponse<T>`) — 원본은 raw 타입 섞임
> - factory 가 모두 `static <T> CommonResponse<T>` — 타입 추론 안전
> - `@JsonInclude(NON_NULL)` 으로 `result` null 이면 JSON 에서 빠짐 (원본 동일)

### 3.1 응답 예시

성공:
```json
{ "code": "OK_001", "status": "OK", "message": "회원가입 성공" }
```

성공 + 데이터:
```json
{
  "code": "OK_001",
  "status": "OK",
  "message": "로그인 성공",
  "result": { "accessToken": "eyJ...", "refreshToken": "eyJ..." }
}
```

실패:
```json
{ "code": "UNAUTH_001", "status": "UNAUTHORIZED", "message": "입력하신 이메일과 비밀번호를 확인해 주세요." }
```

---

## 4. `ResponseCode` enum — 코드 / 메시지 / HTTP status 매핑

```java
// src/main/java/com/example/shop/common/model/enums/ResponseCode.java
@Getter
public enum ResponseCode {

    /* ===== 2xx 성공 ===== */
    OK              (HttpStatus.OK,             "OK_001", "성공"),
    OK_EMPTY        (HttpStatus.OK,             "OK_002", "성공 + 데이터 없음"),

    /* ===== 4xx 클라이언트 ===== */
    // 400 BAD_REQUEST
    BAD_REQUEST             (HttpStatus.BAD_REQUEST, "BADREQ_001", "잘못된 요청입니다."),
    INVALID_INPUT_FORMAT    (HttpStatus.BAD_REQUEST, "BADREQ_002", "입력 형식이 올바르지 않습니다."),
    MISSING_REQUIRED_FIELD  (HttpStatus.BAD_REQUEST, "BADREQ_003", "필수 입력값이 누락되었습니다."),
    DUPLICATE_DATA          (HttpStatus.BAD_REQUEST, "BADREQ_004", "이미 존재하는 데이터입니다."),
    // 401 UNAUTHORIZED
    UNAUTHORIZED            (HttpStatus.UNAUTHORIZED, "UNAUTH_001", "인증이 필요합니다."),
    INVALID_TOKEN           (HttpStatus.UNAUTHORIZED, "UNAUTH_002", "유효하지 않은 토큰입니다."),
    // 403 FORBIDDEN
    FORBIDDEN               (HttpStatus.FORBIDDEN, "FORBID_001", "권한이 없습니다."),
    INSUFFICIENT_PERMISSION (HttpStatus.FORBIDDEN, "FORBID_002", "해당 리소스에 접근할 권한이 없습니다."),
    // 404 NOT_FOUND
    NOT_FOUND               (HttpStatus.NOT_FOUND, "NOTFND_001", "요청한 리소스를 찾을 수 없습니다."),
    USER_NOT_FOUND          (HttpStatus.NOT_FOUND, "NOTFND_002", "사용자를 찾을 수 없습니다."),
    // 409 CONFLICT (개선: 원본은 400 으로 처리하지만, "duplicate / 이미 존재"는 409 가 표준)
    CONFLICT                (HttpStatus.CONFLICT, "CONFLT_001", "리소스 상태 충돌"),
    // 410 GONE
    EXPIRED_AUTH_CODE       (HttpStatus.GONE, "GONE_001", "인증 코드가 만료되었습니다."),
    DELETED_RESOURCE        (HttpStatus.GONE, "GONE_002", "이미 삭제된 리소스입니다."),
    USER_ALREADY_DELETED    (HttpStatus.GONE, "GONE_003", "이미 삭제된 사용자입니다."),
    // 422 UNPROCESSABLE_ENTITY
    AUTH_UNVERIFIED         (HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_001", "인증이 완료되지 않았습니다."),
    ADDITIONAL_INFO_REQUIRED(HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_002", "추가 정보 입력이 필요합니다."),
    SOCIAL_EMAIL_MISSING    (HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_003", "소셜 로그인에서 이메일이 제공되지 않았습니다."),
    // 429 TOO_MANY_REQUESTS (개선 — 원본 없음)
    RATE_LIMIT_EXCEEDED     (HttpStatus.TOO_MANY_REQUESTS, "RATE_001", "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요."),

    /* ===== 5xx 서버 ===== */
    INTERNAL_SERVER_ERROR   (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_001", "서버 내부 오류"),
    DATABASE_ERROR          (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_002", "데이터베이스 오류"),
    EXTERNAL_API_ERROR      (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_003", "외부 API 호출 중 오류"),
    FILE_PROCESSING_ERROR   (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_004", "파일 처리 오류");

    private final HttpStatus httpStatus;
    private final String code;
    private final String message;

    ResponseCode(HttpStatus httpStatus, String code, String message) {
        this.httpStatus = httpStatus;
        this.code = code;
        this.message = message;
    }

    public String code()        { return code; }
    public String message()     { return message; }
    public HttpStatus httpStatus() { return httpStatus; }
}
```

### 4.1 개선 사항

| 원본 (job-answer-be) | 개선 |
| --- | --- |
| `DUPLICATE_DATA` 가 400 | **400 + 별도 409 `CONFLICT`** (REST 표준) |
| `SUFFICIENT_NOT_PERMISSIONS` (오타) | `INSUFFICIENT_PERMISSION` |
| `DDITIONAL_INFO_REQUIRED` (오타) | `ADDITIONAL_INFO_REQUIRED` |
| `APPLE_EMAIL_NOT_PROVIDED` 너무 구체적 | 일반화 `SOCIAL_EMAIL_MISSING` |
| 429 / rate limit 없음 | `RATE_LIMIT_EXCEEDED` 추가 |
| 한국어만 | 영문도 가능 (i18n 은 `message` 만 외부 번역) |

### 4.2 새 코드 추가 정책

- `<카테고리>_<번호 3자리>` — 예: `BADREQ_005`, `UNAUTH_003`
- 같은 도메인 코드는 같은 prefix
- 추가 시 PR 에 변경 이유 명시 / 클라이언트 측에 알림
- **삭제 / 의미 변경 금지** — 한 번 정의된 코드는 deprecated 만 (클라이언트 호환)

---

## 5. `BusinessException` — 도메인 예외

```java
// src/main/java/com/example/shop/common/exception/BusinessException.java
@Getter
public class BusinessException extends RuntimeException {

    private final ResponseCode responseCode;
    private final String customMessage;

    /** ResponseCode 의 기본 메시지를 사용 */
    public BusinessException(ResponseCode responseCode) {
        super(responseCode.message());
        this.responseCode = responseCode;
        this.customMessage = responseCode.message();
    }

    /** ResponseCode + 커스텀 메시지 */
    public BusinessException(ResponseCode responseCode, String customMessage) {
        super(customMessage);
        this.responseCode = responseCode;
        this.customMessage = customMessage;
    }

    /** 코드 없이 메시지만 — 가급적 ResponseCode 명시 권장 */
    public BusinessException(String message) {
        super(message);
        this.responseCode = ResponseCode.INTERNAL_SERVER_ERROR;
        this.customMessage = message;
    }

    public boolean is5xx() {
        return responseCode != null && responseCode.httpStatus().is5xxServerError();
    }
}
```

> **개선 포인트**:
> - 원본은 `responseCode` 가 null 일 수 있음 → 이 코드는 fallback (INTERNAL_SERVER_ERROR) 명시
> - 도메인 코드에서 throw 할 때 **항상 `BusinessException(ResponseCode.X, "메시지")`** 권장

### 5.1 사용 예

```java
// Service
if (userRepo.existsByEmailHash(emailHash)) {
    throw new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입된 이메일입니다.");
}

// 객체 못 찾음
return userRepo.findById(id)
    .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND));

// 권한 없음
if (!order.isOwnedBy(currentUserId)) {
    throw new BusinessException(ResponseCode.FORBIDDEN, "본인의 주문만 조회 가능합니다.");
}
```

---

## 6. `ApiExceptionHandler` — 글로벌

```java
// src/main/java/com/example/shop/common/handler/ApiExceptionHandler.java
@Slf4j
@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ApiExceptionHandler {

    /** 어떤 ApiLogFilter / 모니터링 filter 가 예외 객체를 가져갈 attribute key */
    public static final String EXCEPTION_ATTR = "apiLog.exception";

    // -----------------------------------------------
    // 1) 도메인 예외 — BusinessException
    // -----------------------------------------------
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<CommonResponse<Void>> handleBusiness(BusinessException ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        if (ex.is5xx()) log.error("BIZ 5xx: {}", ex.getMessage(), ex);
        else            log.warn("BIZ: code={}, msg={}", ex.getResponseCode().code(), ex.getMessage());

        var body = CommonResponse.<Void>fail(ex.getResponseCode(), ex.getCustomMessage());
        return ResponseEntity.status(ex.getResponseCode().httpStatus()).body(body);
    }

    // -----------------------------------------------
    // 2) Bean Validation — @Valid, @Validated
    // -----------------------------------------------
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<CommonResponse<Map<String, String>>> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);

        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() == null ? "invalid" : fe.getDefaultMessage(),
                (a, b) -> a
            ));
        log.warn("VALIDATION: fields={}", fieldErrors);

        var body = CommonResponse.<Map<String, String>>builder()
            .code(ResponseCode.INVALID_INPUT_FORMAT.code())
            .status(ResponseCode.INVALID_INPUT_FORMAT)
            .message("입력 형식이 올바르지 않습니다.")
            .result(fieldErrors)
            .build();
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(body);
    }

    // -----------------------------------------------
    // 3) DB 무결성 (unique violation / FK)
    // -----------------------------------------------
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<CommonResponse<Void>> handleDataIntegrity(
            DataIntegrityViolationException ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        log.warn("DB integrity: {}", ex.getMostSpecificCause().getMessage());

        var body = CommonResponse.<Void>fail(ResponseCode.CONFLICT, "리소스 상태 충돌");
        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    // -----------------------------------------------
    // 4) JPA 낙관 락
    // -----------------------------------------------
    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<CommonResponse<Void>> handleOptimisticLock(
            OptimisticLockingFailureException ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        log.warn("Optimistic lock: {}", ex.getMessage());

        var body = CommonResponse.<Void>fail(ResponseCode.CONFLICT,
            "동시 수정이 감지되었습니다. 다시 시도해 주세요.");
        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    // -----------------------------------------------
    // 5) S3 / 파일 처리
    // -----------------------------------------------
    @ExceptionHandler(S3Exception.class)
    public ResponseEntity<CommonResponse<Void>> handleS3(S3Exception ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        log.warn("S3 error: {}", ex.getMessage());
        var body = CommonResponse.<Void>fail(ResponseCode.EXTERNAL_API_ERROR,
            "S3 처리 중 오류가 발생했습니다.");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }

    @ExceptionHandler(IOException.class)
    public ResponseEntity<CommonResponse<Void>> handleIO(IOException ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        log.warn("IO error: {}", ex.getMessage());
        var body = CommonResponse.<Void>fail(ResponseCode.FILE_PROCESSING_ERROR,
            "파일 처리 중 오류가 발생했습니다.");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }

    // -----------------------------------------------
    // 6) 캐치올 — Exception (마지막 안전망)
    // -----------------------------------------------
    @ExceptionHandler(Exception.class)
    public ResponseEntity<CommonResponse<Void>> handleAny(Exception ex, HttpServletRequest req) {
        req.setAttribute(EXCEPTION_ATTR, ex);
        log.error("UNCAUGHT: {}", ex.getMessage(), ex);

        // 운영에선 상세 메시지 노출 X (보안)
        var body = CommonResponse.<Void>fail(ResponseCode.INTERNAL_SERVER_ERROR,
            "서버 내부 오류가 발생했습니다.");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }
}
```

### 6.1 개선 사항

| 원본 | 개선 |
| --- | --- |
| `ex.toString()` 을 message 로 (5xx 에서) | 운영에선 일반 메시지, 디버그 메시지는 로그로만 |
| `MethodArgumentNotValidException` 의 details 없음 | `result` 필드에 field 별 에러 맵 |
| `DataIntegrityViolationException` 핸들러 없음 | 409 매핑 추가 |
| `OptimisticLockingFailureException` 핸들러 없음 | 409 + 재시도 안내 |
| 모든 5xx 로그 `warn` | 5xx 는 `error`, 4xx 는 `warn` |
| `ResponseUtil.getHttpStatus` 호출 | `ex.getResponseCode().httpStatus()` 직접 (간결) |

### 6.2 `@Order(HIGHEST_PRECEDENCE)` 의 의미

- 다른 `@RestControllerAdvice` 가 있어도 이게 먼저 발동
- Spring Security 의 `AccessDeniedHandler` 와는 별도 (Security 는 filter chain 단)

---

## 7. Controller 에서 사용 패턴

```java
@PostMapping("/signup")
@SecurityRequirements()                          // 인증 면제
public ResponseEntity<CommonResponse<SignupResponse>> signup(
    @Valid @RequestBody SignupRequest req
) {
    SignupResponse data = signupService.signup(req);
    var body = CommonResponse.success(ResponseCode.OK, data, "회원가입 성공");
    return ResponseEntity.status(ResponseCode.OK.httpStatus()).body(body);
}
```

또는 `ResponseEntity` 없이 (status 가 ResponseCode 와 동기화):

```java
@PostMapping("/signup")
@ResponseStatus(HttpStatus.OK)                   // 명시
public CommonResponse<SignupResponse> signup(@Valid @RequestBody SignupRequest req) {
    return CommonResponse.success(ResponseCode.OK, signupService.signup(req), "회원가입 성공");
}
```

> **권장**: 정상 응답은 `ResponseEntity` 없이 단순 반환. 실패는 throw `BusinessException` → handler 가 status 매핑. Controller 가 간결.

---

## 8. `ResponseUtil` (선택 — 호환용)

```java
public final class ResponseUtil {
    private ResponseUtil() {}

    public static HttpStatus getHttpStatus(CommonResponse<?> response) {
        if (response.getStatus() == null) return HttpStatus.INTERNAL_SERVER_ERROR;
        return response.getStatus().httpStatus();
    }
}
```

→ 기존 코드와의 호환을 위해 유지. 새 코드는 `ex.getResponseCode().httpStatus()` 직접 사용.

---

## 9. 로깅 / 감사 — ApiLogFilter 와 연동

`ApiExceptionHandler.EXCEPTION_ATTR` 로 예외 객체를 request attribute 에 저장. 별도 `ApiLogFilter` 가 응답 후 picked up:

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)            // Exception handler 보다 뒤
public class ApiLogFilter extends OncePerRequestFilter {

    private final ApiLogRepository logs;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        var start = Instant.now();
        try {
            chain.doFilter(req, res);
        } finally {
            var ex = (Exception) req.getAttribute(ApiExceptionHandler.EXCEPTION_ATTR);
            logs.record(new ApiLogRow(
                req.getMethod(),
                req.getRequestURI(),
                res.getStatus(),
                Duration.between(start, Instant.now()).toMillis(),
                ex == null ? null : ex.getClass().getSimpleName(),
                ex == null ? null : ex.getMessage()
            ));
        }
    }
}
```

→ 모든 API 호출이 access log 보다 풍부한 메타 (예외 객체) 를 가짐. 운영 대시보드 / 알람 (5xx 비율) 에 활용.

---

## 10. 테스트

```java
@Test
void business_exception_returns_correct_code_and_status() {
    var ex = new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입");
    ResponseEntity<CommonResponse<Void>> res = handler.handleBusiness(ex, mockRequest());

    assertThat(res.getStatusCode().value()).isEqualTo(400);
    assertThat(res.getBody().getCode()).isEqualTo("BADREQ_004");
    assertThat(res.getBody().getMessage()).isEqualTo("이미 가입");
}

@Test
void validation_returns_field_errors_in_result() {
    var ex = mockValidationException(Map.of("email", "must be email"));
    var res = handler.handleValidation(ex, mockRequest());

    assertThat(res.getStatusCode().value()).isEqualTo(422);
    assertThat(res.getBody().getResult()).containsEntry("email", "must be email");
}

@Test
void uncaught_exception_does_not_leak_internal_message() {
    var ex = new RuntimeException("DB password = abc123");
    var res = handler.handleAny(ex, mockRequest());

    assertThat(res.getBody().getMessage()).isEqualTo("서버 내부 오류가 발생했습니다.");
    // 내부 메시지 노출 X
}
```

---

## 11. 운영 체크리스트

- [ ] 모든 Controller 에 try-catch 없음 — 예외는 handler 가 처리
- [ ] 모든 도메인 예외가 `BusinessException(ResponseCode.X, ...)` 형태
- [ ] `Exception` 캐치올은 운영에서 메시지 노출 X
- [ ] 5xx 비율 모니터링 (APM / Prometheus)
- [ ] 새 `ResponseCode` 추가 시 클라이언트 측에 알림
- [ ] `ApiLogFilter` 가 모든 응답을 기록
- [ ] `ApiExceptionHandler` 는 `@Order(HIGHEST_PRECEDENCE)`
- [ ] Spring Security 의 401/403 은 별도 (filter 단) — `JwtAuthenticationEntryPoint` ([[security-config]])

---

## 12. 함정 모음

### 함정 1 — `ex.toString()` 을 응답 메시지로
스택 / DB 패스워드 / 내부 path 노출. **응답에는 일반 메시지, 디버그는 로그**.

### 함정 2 — `responseCode` 이 nullable
`is5xx()` 에서 NPE. **항상 ResponseCode 매핑 후 throw**.

### 함정 3 — Validation 의 details 없음
사용자가 어느 필드가 틀렸는지 모름. **`result` 필드에 field 별 에러**.

### 함정 4 — `DataIntegrityViolationException` 안 잡음
500 으로 빠짐. 실제로는 409 (unique / FK 충돌). **별도 핸들러**.

### 함정 5 — Spring Security 의 401/403 도 `@RestControllerAdvice` 가 처리할 거라 기대
filter chain 단에서 발동 — handler 가 못 잡음. **`AuthenticationEntryPoint`, `AccessDeniedHandler` 별도**.

### 함정 6 — `@Order` 없음
다른 advice 와 충돌 시 어떤 게 발동할지 모름. **`@Order(HIGHEST_PRECEDENCE)`**.

### 함정 7 — `BindingResult` 의 field 가 중복
같은 필드에 여러 에러가 붙음. `(a, b) -> a` 로 첫 에러만 선택 또는 list 로.

### 함정 8 — `result` 에 sensitive data
예외 메시지에 카드번호 / 패스워드 등 들어가지 않게 검토.

### 함정 9 — 모든 응답에 `result` 강제
성공 응답에 `result: null` 표시 = 클라가 항상 `if (result)` 처리. **`@JsonInclude(NON_NULL)`**.

### 함정 10 — code 의 의미 변경
한 번 정의된 코드의 HTTP status / 의미 바뀌면 클라이언트 깨짐. **deprecated 만, 삭제 X**.

---

## 13. 관련

- [[security-config]] (다음) — Security filter chain + Cors + JwtAuthenticationEntryPoint
- [[../api-docs/swagger-springdoc]] — `@ApiResponse` 에 `ResponseCode` 마다 응답 예시
- [[../api-design/api-design|↗ API 레시피]] — 본 패턴을 쓰는 8 레시피
- [[../pitfalls/transaction-pitfalls]] (예정)
