---
title: "auth §11 — 휴대폰 인증 구현 (NCP SENS / AlimTalk)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - phone-verification
  - sms
---

# auth §11 — 휴대폰 인증 구현 (NCP SENS / AlimTalk)

**[[signup|↑ hub]]**  ·  ← [[email-verification-impl]]  ·  → [[login-impl]]

> 한국 SaaS 표준. 가입 전 / 가입 후 (변경) 모두 사용 가능한 흐름.
> 도구 선택: [[design-decisions#6 SMS 발송 도구]] 참고.

---

## 1. API spec

```
POST /api/v1/auth/verify/phone/request   { phone }
   ↓ SMS / AlimTalk 발송
POST /api/v1/auth/verify/phone/confirm   { phone, code }
   ↓ 통과 시 phoneAuthToken 발급
[가입 / 패스워드 변경 등 본인 확인 필요 endpoint] body 에 phoneAuthToken 포함
```

### 1.1 Request — 인증번호 요청

```http
POST /api/v1/auth/verify/phone/request
Content-Type: application/json

{ "phone": "010-1234-5678" }
```

### 1.2 Response

```json
{
  "code": "OK_001",
  "status": "OK",
  "message": "인증번호 발송 완료",
  "result": {
    "requestId": "01HZ-REQ-...",
    "expiresInSeconds": 180
  }
}
```

### 1.3 Confirm

```http
POST /api/v1/auth/verify/phone/confirm
{ "phone": "010-1234-5678", "code": "123456" }
```

```json
{
  "result": {
    "phoneAuthToken": "01HZ-TOKEN-...",
    "expiresInSeconds": 600
  }
}
```

→ 클라가 다음 단계 (signup / change phone / step-up) 에 `phoneAuthToken` 헤더 또는 body 로 전달. 서버가 verifiedAt 확인.

---

## 2. 비기능

| 항목 | 정책 |
| --- | --- |
| 코드 형식 | 6자리 숫자 |
| 만료 | 인증번호 3분 / phoneAuthToken 10분 |
| Rate limit | 같은 phone 5건/시간 + 같은 IP 20건/시간 |
| 실패 시 | 5회 틀리면 잠금 (해당 코드 무효 — 재발송 필요) |
| 동일 phone 동시 인증 | 마지막 발송만 유효 (이전 invalidate) |
| 저장 | Redis (인증번호 TTL) + 토큰 (선택적 RDB) |

---

## 3. 도메인

```java
// domain/auth/PhoneVerification.java
public final class PhoneVerification {

    public enum Status { PENDING, VERIFIED, EXPIRED, REVOKED }

    private final PhoneVerificationId id;
    private final PhoneNumber phone;
    private final String codeHash;                        // SHA-256(6 digit)
    private final Instant issuedAt;
    private final Instant expiresAt;
    private int attempts;
    private Status status;

    private static final int MAX_ATTEMPTS = 5;

    public static PhoneVerification request(PhoneVerificationId id, PhoneNumber phone,
                                             String code, Instant now, Duration ttl) {
        var hash = sha256Hex(code);
        return new PhoneVerification(id, phone, hash, now, now.plus(ttl), 0, Status.PENDING);
    }

    public void tryConfirm(String code, Instant now) {
        if (status != Status.PENDING)
            throw new BusinessException(ResponseCode.INVALID_TOKEN, "유효하지 않은 인증 요청");
        if (now.isAfter(expiresAt))
            throw new BusinessException(ResponseCode.EXPIRED_AUTH_CODE, "인증번호 만료");
        if (attempts >= MAX_ATTEMPTS) {
            status = Status.REVOKED;
            throw new BusinessException(ResponseCode.FORBIDDEN, "시도 횟수 초과");
        }
        attempts++;
        if (!sha256Hex(code).equals(codeHash)) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "인증번호 불일치");
        }
        status = Status.VERIFIED;
    }
    // getters
}

public record PhoneNumber(String value) {
    private static final Pattern KR = Pattern.compile("^01[0-9]-?[0-9]{3,4}-?[0-9]{4}$");
    public PhoneNumber {
        if (value == null || !KR.matcher(value).matches())
            throw new IllegalArgumentException("invalid phone: " + value);
    }
    public String normalized() {       // "010-1234-5678" → "01012345678"
        return value.replaceAll("-", "");
    }
}
```

---

## 4. DB / Redis

### 4.1 Redis (인증번호 TTL — 가벼움)

```
Key:   phone:verify:{phone-normalized}
Value: JSON { id, codeHash, attempts, issuedAt, status }
TTL:   3분
```

### 4.2 PhoneAuthToken (Redis 또는 RDB)

```
Key:   phone:auth-token:{token}
Value: JSON { phone, verifiedAt }
TTL:   10분
```

```sql
-- 또는 영속 (audit 필요 시)
CREATE TABLE phone_verifications (
    id           CHAR(26) PRIMARY KEY,
    phone        VARCHAR(20) NOT NULL,
    code_hash    CHAR(64) NOT NULL,
    status       VARCHAR(20) NOT NULL,
    attempts     INTEGER NOT NULL DEFAULT 0,
    issued_at    TIMESTAMPTZ NOT NULL,
    expires_at   TIMESTAMPTZ NOT NULL,
    verified_at  TIMESTAMPTZ
);
CREATE INDEX ix_phone_verif_phone_status ON phone_verifications (phone, status, issued_at DESC);
```

---

## 5. SMS Client (NCP SENS)

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-webflux")    // WebClient
```

```yaml
app:
  sms:
    ncp-sens:
      base-url: https://sens.apigw.ntruss.com
      service-id: ${NCP_SERVICE_ID}
      access-key: ${NCP_ACCESS_KEY}
      secret-key: ${NCP_SECRET_KEY}
      sender: "025550000"               # 발신 번호 등록 필요
```

```java
// infrastructure/external/sms/NcpSensSmsClient.java
@Component
@RequiredArgsConstructor
@Slf4j
public class NcpSensSmsClient implements SmsClient {

    private final RestClient http;
    @Value("${app.sms.ncp-sens.service-id}") String serviceId;
    @Value("${app.sms.ncp-sens.access-key}") String accessKey;
    @Value("${app.sms.ncp-sens.secret-key}") String secretKey;
    @Value("${app.sms.ncp-sens.sender}") String sender;

    @CircuitBreaker(name = "sms-ncp", fallbackMethod = "fallback")
    @Retry(name = "sms-ncp")
    @Override
    public SmsResult send(String to, String content) {
        var timestamp = String.valueOf(System.currentTimeMillis());
        var url = "/sms/v2/services/" + serviceId + "/messages";
        var signature = makeSignature("POST", url, timestamp);

        var body = Map.of(
            "type", "SMS",
            "from", sender,
            "content", content,
            "messages", List.of(Map.of("to", to.replaceAll("-", "")))
        );
        try {
            var response = http.post()
                .uri(url)
                .header("Content-Type", "application/json")
                .header("x-ncp-apigw-timestamp", timestamp)
                .header("x-ncp-iam-access-key", accessKey)
                .header("x-ncp-apigw-signature-v2", signature)
                .body(body)
                .retrieve()
                .body(Map.class);
            return new SmsResult(true, (String) response.get("requestId"), null);
        } catch (HttpStatusCodeException e) {
            log.error("NCP SENS error: {}", e.getResponseBodyAsString());
            return new SmsResult(false, null, e.getStatusCode().toString());
        }
    }

    private SmsResult fallback(String to, String content, Throwable t) {
        return new SmsResult(false, null, "SMS 발송 일시 불가: " + t.getMessage());
    }

    private String makeSignature(String method, String url, String timestamp) {
        try {
            var message = method + " " + url + "\n" + timestamp + "\n" + accessKey;
            var mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secretKey.getBytes(UTF_8), "HmacSHA256"));
            return Base64.getEncoder().encodeToString(mac.doFinal(message.getBytes(UTF_8)));
        } catch (Exception e) { throw new IllegalStateException(e); }
    }
}

public interface SmsClient {
    SmsResult send(String to, String content);
    record SmsResult(boolean ok, String externalRequestId, String errorMessage) {}
}
```

→ **CircuitBreaker / Retry / Fallback** 으로 SMS 외부 장애 대응.

---

## 6. UseCase — Request / Confirm

```java
// application/auth/RequestPhoneVerificationUseCase.java
@Service
@RequiredArgsConstructor
@Slf4j
public class RequestPhoneVerificationUseCase {

    private final PhoneVerificationRedisRepository repo;
    private final SmsClient sms;
    private final SmsRateLimiter rateLimiter;
    private final IdGenerator ids;
    private final Clock clock;
    private final SecureRandom random = new SecureRandom();

    @Value("${app.phone-verify.ttl:PT3M}") Duration ttl;

    public RequestResult handle(PhoneNumber phone, String clientIp) {
        // Rate limit
        rateLimiter.checkPhone(phone);
        rateLimiter.checkIp(clientIp);

        // 같은 phone 의 이전 PENDING 무효
        repo.deleteByPhone(phone);

        // 6자리 코드
        var code = String.format("%06d", random.nextInt(1_000_000));
        var verification = PhoneVerification.request(
            new PhoneVerificationId(ids.next()), phone, code,
            Instant.now(clock), ttl
        );
        repo.save(verification);

        // SMS 발송 (비동기 옵션)
        var result = sms.send(phone.normalized(),
            "[Shop] 인증번호 [" + code + "]를 입력해 주세요. (유효시간 3분)");

        if (!result.ok()) {
            log.warn("SMS 발송 실패: phone={}, error={}", phone, result.errorMessage());
            throw new BusinessException(ResponseCode.EXTERNAL_API_ERROR,
                "SMS 발송 실패 — 잠시 후 다시 시도해 주세요.");
        }

        return new RequestResult(verification.id().value(), (int) ttl.toSeconds());
    }

    public record RequestResult(String requestId, int expiresInSeconds) {}
}
```

```java
// application/auth/ConfirmPhoneVerificationUseCase.java
@Service
@RequiredArgsConstructor
public class ConfirmPhoneVerificationUseCase {

    private final PhoneVerificationRedisRepository verifications;
    private final PhoneAuthTokenRepository authTokens;
    private final IdGenerator ids;
    private final Clock clock;
    @Value("${app.phone-verify.auth-token-ttl:PT10M}") Duration tokenTtl;

    public ConfirmResult handle(PhoneNumber phone, String code) {
        var verification = verifications.findActiveByPhone(phone)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND,
                "인증 요청을 찾을 수 없습니다. 다시 발송해 주세요."));

        verification.tryConfirm(code, Instant.now(clock));      // 도메인이 모든 검증
        verifications.save(verification);

        // 통과 시 — phoneAuthToken 발급 (다음 단계에서 사용)
        var rawToken = ids.next() + ULID.random().substring(0, 6);
        authTokens.save(new PhoneAuthToken(rawToken, phone, Instant.now(clock), tokenTtl));

        return new ConfirmResult(rawToken, (int) tokenTtl.toSeconds());
    }

    public record ConfirmResult(String phoneAuthToken, int expiresInSeconds) {}
}
```

---

## 7. PhoneAuthToken 활용 — Signup / Step-up

가입 시:
```http
POST /api/v1/auth/signup
{
  "email": "alice@x.com",
  "password": "...",
  "name": "Alice",
  "phone": "010-1234-5678",
  "phoneAuthToken": "01HZ-TOKEN-...",       ← 인증 통과 증명
  "termsAgreed": true
}
```

서버:
```java
// SignupUseCase 의 시작 부분에서 검증
if (cmd.phoneAuthToken() != null) {
    var token = phoneAuthTokens.find(cmd.phoneAuthToken())
        .orElseThrow(() -> new BusinessException(ResponseCode.AUTH_UNVERIFIED,
            "휴대폰 인증이 완료되지 않았습니다."));
    if (!token.phone().equals(new PhoneNumber(cmd.phone())))
        throw new BusinessException(ResponseCode.AUTH_UNVERIFIED, "인증된 휴대폰과 일치하지 않음");
    phoneAuthTokens.delete(cmd.phoneAuthToken());     // 1회용
}
```

→ 본인 확인 필요한 다른 endpoint (패스워드 변경 / 휴대폰 번호 변경 / 환불 등) 도 같은 패턴.

---

## 8. Rate Limiter

```java
@Component
@RequiredArgsConstructor
public class SmsRateLimiter {

    private final RateLimiter limiter;

    public void checkPhone(PhoneNumber phone) {
        var key = "rl:phone-sms:" + phone.normalized();
        var result = limiter.tryConsume(key, RateLimitPolicy.of(5, Duration.ofHours(1)));
        if (!result.allowed())
            throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED,
                "휴대폰 인증 요청 한도 초과 (1시간 5회). 잠시 후 다시 시도해 주세요.");
    }

    public void checkIp(String ip) {
        var key = "rl:phone-sms-ip:" + ip;
        var result = limiter.tryConsume(key, RateLimitPolicy.of(20, Duration.ofHours(1)));
        if (!result.allowed())
            throw new BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED, "요청 한도 초과");
    }
}
```

[[../rate-limiting]] 의 Bucket4j + Redis 활용.

---

## 9. Controller

```java
@Tag(name = "휴대폰 인증")
@RestController
@RequestMapping("/api/v1/auth/verify/phone")
@RequiredArgsConstructor
public class PhoneVerificationController {

    private final RequestPhoneVerificationUseCase request;
    private final ConfirmPhoneVerificationUseCase confirm;

    @Operation(summary = "휴대폰 인증번호 발송")
    @SecurityRequirements()
    @PostMapping("/request")
    public ResponseEntity<CommonResponse<RequestPhoneVerificationUseCase.RequestResult>> request(
        @Valid @RequestBody PhoneRequestDto req,
        HttpServletRequest http
    ) {
        var ip = ClientIpUtil.resolveClientIp(http);
        var result = request.handle(new PhoneNumber(req.phone()), ip);
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, result, "발송 완료"));
    }

    @Operation(summary = "인증번호 확인")
    @SecurityRequirements()
    @PostMapping("/confirm")
    public ResponseEntity<CommonResponse<ConfirmPhoneVerificationUseCase.ConfirmResult>> confirm(
        @Valid @RequestBody PhoneConfirmDto req
    ) {
        var result = confirm.handle(new PhoneNumber(req.phone()), req.code());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, result, "인증 완료"));
    }
}

public record PhoneRequestDto(@NotBlank @Pattern(regexp = "^01[0-9]-?[0-9]{3,4}-?[0-9]{4}$") String phone) {}
public record PhoneConfirmDto(
    @NotBlank @Pattern(regexp = "^01[0-9]-?[0-9]{3,4}-?[0-9]{4}$") String phone,
    @NotBlank @Pattern(regexp = "^\\d{6}$") String code
) {}
```

---

## 10. AlimTalk fallback (옵션)

NCP SENS 외에 **카카오 비즈메시지 AlimTalk** 사용 시 — 카카오톡 등록 사용자에게 알림톡 발송 (실패 시 SMS fallback).

```java
@Component
public class HybridSmsClient implements SmsClient {

    private final AlimTalkClient alimTalk;
    private final NcpSensSmsClient sms;

    @Override
    public SmsResult send(String to, String content) {
        var alimTalkResult = alimTalk.send(to, "AUTH_CODE", Map.of("code", extractCode(content)));
        if (alimTalkResult.ok()) return alimTalkResult;

        // AlimTalk 실패 (카카오톡 미사용자 / 친구 X) → SMS fallback
        return sms.send(to, content);
    }
}
```

→ 비용 ↓ (AlimTalk 가 SMS 보다 저렴). UX ↑ (카카오톡으로 알림).

---

## 11. 함정 모음

### 함정 1 — 인증번호를 평문 Redis 저장
같은 사람이 redis-cli 보면 인증 가능. **SHA-256 hash 저장**.

### 함정 2 — phone 정규화 없음
"010-1234-5678" 과 "01012345678" 가 다른 키. **항상 정규화** (하이픈 제거).

### 함정 3 — Rate limit 없음
같은 phone 으로 100 번 발송 = SMS 비용 폭발 + 사용자 피해. **시간당 5건 제한**.

### 함정 4 — 코드 brute force
6자리 = 100만. 무한 시도면 1분 안에 뚫림. **5회 실패 = 잠금**.

### 함정 5 — phoneAuthToken 만료 안 둠
영구 토큰 = 한 번 인증 후 영원히 사용 가능. **10분 TTL**.

### 함정 6 — 토큰 재사용
phoneAuthToken 으로 signup + 패스워드 변경 둘 다 가능 = 위험. **1회용** (사용 후 삭제).

### 함정 7 — SMS 발송 실패 시 에러 노출
NCP error code 그대로 사용자에 노출. **일반화** ("SMS 발송 실패").

### 함정 8 — 동시성 — 같은 phone 동시 2 요청
둘 다 6자리 코드 발송 → 사용자 혼동. **마지막만 유효** (이전 invalidate).

### 함정 9 — 외부 SMS provider 다운 시 전체 가입 중단
**Circuit Breaker** + 사용자에 명확한 메시지.

### 함정 10 — 본인 인증과 혼동
SMS 6자리 = "휴대폰 보유" 인증. **본인 (실명) 인증은 별도** (PASS / NICE).

---

## 12. 운영 체크리스트

- [ ] NCP SENS service-id / access-key / secret-key — vault
- [ ] 발신번호 등록 (NCP / 카카오)
- [ ] AlimTalk 템플릿 등록 + 승인
- [ ] Rate limit (phone / IP)
- [ ] SMS 발송 성공율 모니터 (Prometheus)
- [ ] 비용 모니터 (월 SMS 건수)
- [ ] phone 정규화 통일 (DB / Redis / 외부 호출 모두)
- [ ] Circuit Breaker 적용
- [ ] AlimTalk 실패 → SMS fallback (옵션)

---

## 13. 본인 인증 (PASS / NICE) 통합 — 금융·의료 도메인

본 레시피는 일반 휴대폰 보유 확인. 실명 인증이 필요하면:

```
POST /api/v1/auth/verify/identity/init
   ↓
NICE / PASS 인증 페이지로 redirect
   ↓
사용자 본인 인증 (통신 3사)
   ↓
callback URL 로 결과 받음
   ↓
서버: NICE 응답에서 이름·생년월일·휴대폰·CI/DI 추출
   ↓
user 에 저장 (CI 는 unique 보장)
```

→ 별도 통합 문서 필요. NICE 가입 + 가맹점 가입 + 키 발급 필요.

---

## 14. 관련

- [[signup|↑ hub]]
- [[design-decisions#6 SMS 발송 도구]]
- [[email-verification-impl]] — 비슷한 토큰 구조
- [[signup-impl]] — phoneAuthToken 통합 흐름
- [[../rate-limiting]] — Bucket4j
