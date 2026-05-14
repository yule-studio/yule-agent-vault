---
title: "2FA / TOTP (Google Authenticator) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - 2fa
  - totp
---

# 2FA / TOTP (Google Authenticator) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 6238 TOTP / QR / recovery codes |

**[[api-design|↑ api-design hub]]**

> 📐 **ORM**: §6 는 JPA Adapter sketch. 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가

```
POST /api/v1/auth/2fa/enroll             # secret 생성 + QR 반환 (Authenticator 등록 전)
POST /api/v1/auth/2fa/enable             # TOTP 코드 확인 후 활성화 + recovery codes 반환
POST /api/v1/auth/2fa/disable            # 비활성화 (현재 패스워드 + TOTP 필요)
POST /api/v1/auth/2fa/verify             # 로그인 후 2FA 검증 (login 흐름에 끼움)
POST /api/v1/auth/2fa/recovery           # recovery code 로 2FA 우회 (분실 시)
```

### 1.1 로그인 흐름 with 2FA

```
1. POST /auth/login (email + password)
   → 2FA off: access + refresh 발급
   → 2FA on:  "2FA 필요" 응답 + 임시 토큰 (2fa_pending, 5분)
2. POST /auth/2fa/verify (임시 토큰 + 6-digit code)
   → 검증 통과 → 정식 access + refresh 발급
```

### 1.2 비기능

- **RFC 6238 TOTP** — 30초 윈도우 / SHA-1 / 6 digit
- **clock skew 허용** — 직전 / 현재 / 다음 윈도우 (±1) 까지 OK
- **recovery code** — 10개, 단일 사용, hash 저장
- **secret 은 사용자별** — AES-GCM 으로 DB 컬럼 암호화
- **임시 토큰 (2fa_pending)** — short TTL JWT, 별도 type
- TOTP 코드 brute force 방어 — 5회 실패 후 lock

---

## 2. 도메인

```java
// domain/auth/UserTwoFactor.java
public final class UserTwoFactor {

    public enum Status { DISABLED, PENDING, ENABLED }

    private final UserId userId;
    private byte[] secretEncrypted;          // AES-GCM 으로 암호화된 TOTP secret
    private Status status;
    private final List<RecoveryCode> recoveryCodes;
    private int failedAttempts;
    private Instant lockedUntil;
    private Instant enabledAt;

    public boolean isEnabled() { return status == Status.ENABLED; }

    public void verify(int code, TotpVerifier verifier, Instant now) {
        if (lockedUntil != null && now.isBefore(lockedUntil)) {
            throw new BusinessException(ResponseCode.FORBIDDEN,
                "2FA 가 잠겨 있습니다. " + lockedUntil + " 이후 시도해 주세요.");
        }
        if (status != Status.ENABLED) {
            throw new BusinessException(ResponseCode.BAD_REQUEST, "2FA 가 활성화되지 않았습니다.");
        }
        if (!verifier.verify(secretEncrypted, code, now)) {
            failedAttempts++;
            if (failedAttempts >= 5) {
                lockedUntil = now.plus(Duration.ofMinutes(15));
                failedAttempts = 0;
            }
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 인증 코드");
        }
        failedAttempts = 0;
        lockedUntil = null;
    }

    public RecoveryCode consumeRecovery(String rawCode) {
        var hash = sha256Hex(rawCode);
        var match = recoveryCodes.stream()
            .filter(r -> r.tokenHash().equals(hash) && !r.used()).findFirst()
            .orElseThrow(() -> new BusinessException(ResponseCode.UNAUTHORIZED,
                "유효하지 않은 복구 코드"));
        match.markUsed(Instant.now());
        return match;
    }
    // ...
}

public record RecoveryCode(String tokenHash, boolean used, Instant usedAt) {
    public RecoveryCode markUsed(Instant now) {
        return new RecoveryCode(tokenHash, true, now);
    }
}
```

---

## 3. DB 스키마

```sql
-- V70__create_user_two_factor.sql
CREATE TABLE user_two_factor (
  user_id          CHAR(26) PRIMARY KEY REFERENCES users(id),
  secret_encrypted BYTEA NOT NULL,                  -- AES-GCM 암호화
  status           VARCHAR(20) NOT NULL,
  failed_attempts  INTEGER NOT NULL DEFAULT 0,
  locked_until     TIMESTAMPTZ,
  enabled_at       TIMESTAMPTZ,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_recovery_codes (
  id           CHAR(26) PRIMARY KEY,
  user_id      CHAR(26) NOT NULL REFERENCES users(id),
  token_hash   CHAR(64) NOT NULL,                   -- SHA-256(raw recovery code)
  used         BOOLEAN NOT NULL DEFAULT false,
  used_at      TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_user_recovery_codes_user ON user_recovery_codes (user_id);
```

---

## 4. TOTP 라이브러리 + verifier

```kotlin
// build.gradle.kts
implementation("com.warrenstrange:googleauth:1.5.0")     // GoogleAuthenticator
implementation("com.google.zxing:core:3.5.3")             // QR
implementation("com.google.zxing:javase:3.5.3")
```

```java
// infrastructure/security/totp/TotpService.java
@Component
@RequiredArgsConstructor
public class TotpService implements TotpVerifier {

    private final SecretEncryption secretEncryption;     // AES-GCM (KMS)

    private static final GoogleAuthenticator GAUTH = new GoogleAuthenticator(
        new GoogleAuthenticatorConfig.GoogleAuthenticatorConfigBuilder()
            .setWindowSize(1)                            // ±1 windows (90 sec window)
            .setCodeDigits(6)
            .setTimeStepSizeInMillis(30_000)
            .build()
    );

    /** secret 생성 (base32 string). 등록 시점에만 사용. */
    public String generateSecret() {
        return GAUTH.createCredentials().getKey();
    }

    /** otpauth:// URI — QR 생성 입력 */
    public String buildOtpAuthUri(String userEmail, String issuer, String secret) {
        return String.format(
            "otpauth://totp/%s:%s?secret=%s&issuer=%s",
            URLEncoder.encode(issuer, StandardCharsets.UTF_8),
            URLEncoder.encode(userEmail, StandardCharsets.UTF_8),
            secret,
            URLEncoder.encode(issuer, StandardCharsets.UTF_8)
        );
    }

    public byte[] generateQrPng(String otpAuthUri, int size) throws Exception {
        var bitMatrix = new QRCodeWriter().encode(otpAuthUri, BarcodeFormat.QR_CODE, size, size);
        var baos = new ByteArrayOutputStream();
        MatrixToImageWriter.writeToStream(bitMatrix, "PNG", baos);
        return baos.toByteArray();
    }

    @Override
    public boolean verify(byte[] secretEncrypted, int code, Instant now) {
        var rawSecret = secretEncryption.decrypt(secretEncrypted);
        return GAUTH.authorize(rawSecret, code);
    }
}
```

---

## 5. UseCase

### 5.1 Enroll (등록 — pending 상태)

```java
@Service
@RequiredArgsConstructor
public class Enroll2FAUseCase {

    private final UserTwoFactorRepository repo;
    private final TotpService totp;
    private final SecretEncryption secretEnc;
    private final Clock clock;

    @Transactional
    public EnrollResponse handle(UserId userId, String email) {
        var existing = repo.findByUserId(userId);
        if (existing.isPresent() && existing.get().status() == UserTwoFactor.Status.ENABLED) {
            throw new BusinessException(ResponseCode.CONFLICT, "2FA 가 이미 활성화되어 있습니다.");
        }

        var secret = totp.generateSecret();
        var encrypted = secretEnc.encrypt(secret.getBytes(StandardCharsets.UTF_8));

        var tf = existing.orElseGet(() ->
            new UserTwoFactor(userId, encrypted, UserTwoFactor.Status.PENDING,
                List.of(), 0, null, null));
        tf.replaceSecret(encrypted);                // pending 상태로 재설정
        repo.save(tf);

        var uri = totp.buildOtpAuthUri(email, "Shop", secret);
        byte[] qrPng;
        try { qrPng = totp.generateQrPng(uri, 200); }
        catch (Exception e) { throw new BusinessException(ResponseCode.INTERNAL_SERVER_ERROR, "QR 생성 실패"); }

        return new EnrollResponse(uri, "data:image/png;base64," + Base64.getEncoder().encodeToString(qrPng));
    }

    public record EnrollResponse(String otpAuthUri, String qrDataUrl) {}
}
```

### 5.2 Enable (확정 + recovery codes)

```java
@Service
@RequiredArgsConstructor
public class Enable2FAUseCase {

    private final UserTwoFactorRepository repo;
    private final TotpService totp;
    private final IdGenerator ids;
    private final SecureRandom random = new SecureRandom();

    @Transactional
    public List<String> handle(UserId userId, int code) {
        var tf = repo.findByUserId(userId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "2FA 등록 정보가 없습니다."));
        if (tf.status() != UserTwoFactor.Status.PENDING) {
            throw new BusinessException(ResponseCode.CONFLICT, "이미 활성화된 2FA");
        }
        if (!totp.verify(tf.secretEncrypted(), code, Instant.now())) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "TOTP 코드가 유효하지 않습니다.");
        }

        // 10개 recovery codes 생성 + hash 저장
        var rawCodes = new ArrayList<String>();
        var hashedCodes = new ArrayList<RecoveryCode>();
        for (int i = 0; i < 10; i++) {
            var raw = randomCode();              // "1234-5678-90AB" 같은 형식
            rawCodes.add(raw);
            hashedCodes.add(new RecoveryCode(sha256Hex(raw), false, null));
        }
        tf.activate(hashedCodes);
        repo.save(tf);

        return rawCodes;     // 클라에 한 번만 노출 — 사용자가 종이에 적도록
    }

    private String randomCode() {
        var bytes = new byte[10];
        random.nextBytes(bytes);
        var hex = HexFormat.of().formatHex(bytes).toUpperCase();
        return hex.substring(0, 4) + "-" + hex.substring(4, 8) + "-" + hex.substring(8, 12);
    }
}
```

### 5.3 Login 흐름 통합

login 성공 시 2FA enabled 면 임시 토큰 (5분 TTL, type=`2FA_PENDING`) 발급. 정식 토큰은 verify 후.

```java
// LoginUseCase 의 끝부분 보강
if (twoFactorRepo.isEnabled(user.id())) {
    var pendingToken = jwt.generate2FAPendingToken(user.id(), Duration.ofMinutes(5));
    return new LoginResult(null, null, null, pendingToken, true);  // need2FA = true
}
// else 정상 access + refresh 발급
```

### 5.4 Verify2FAUseCase

```java
@Service
@RequiredArgsConstructor
public class Verify2FAUseCase {

    private final JwtTokenProvider jwt;
    private final UserTwoFactorRepository repo;
    private final UserRepository users;
    private final TotpService totp;
    private final RefreshTokenService rtService;
    private final Clock clock;

    @Transactional
    public LoginResult handle(String pendingToken, int code, String device, String ip) {
        if (!jwt.is2FAPending(pendingToken)) {
            throw new BusinessException(ResponseCode.UNAUTHORIZED, "유효하지 않은 2FA 토큰");
        }
        var userId = new UserId(jwt.getUserId(pendingToken).toString());
        var tf = repo.findByUserId(userId).orElseThrow();
        var user = users.findById(userId).orElseThrow();

        tf.verify(code, totp, Instant.now(clock));
        repo.save(tf);

        // 정식 access + refresh 발급
        var access = jwt.generateAccessToken(user.email().value(), user.id().value(),
                                             user.role().name(), UUID.randomUUID().toString());
        var rt = rtService.issue(user.id(), device, ip);
        return new LoginResult(access, rt.raw(), rt.expiresAt(), null, false);
    }
}
```

---

## 6. Controller

```java
@Tag(name = "2FA")
@RestController
@RequestMapping("/api/v1/auth/2fa")
@RequiredArgsConstructor
public class TwoFactorController {

    private final Enroll2FAUseCase enroll;
    private final Enable2FAUseCase enable;
    private final Disable2FAUseCase disable;
    private final Verify2FAUseCase verify;
    private final RecoveryUseCase recovery;

    @Operation(summary = "2FA 등록 — secret 생성 + QR")
    @PostMapping("/enroll")
    public ResponseEntity<CommonResponse<Enroll2FAUseCase.EnrollResponse>> enroll(Authentication auth) {
        var user = (User) auth.getPrincipal();
        var res = enroll.handle(user.id(), user.email().value());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, res, "2FA 등록 — TOTP 코드 확인 필요"));
    }

    @Operation(summary = "2FA 활성화 — TOTP 코드 확인 + recovery codes 반환")
    @PostMapping("/enable")
    public ResponseEntity<CommonResponse<Map<String, List<String>>>> enable(
        @Valid @RequestBody Enable2FARequest req, Authentication auth
    ) {
        var user = (User) auth.getPrincipal();
        var codes = enable.handle(user.id(), req.code());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("recoveryCodes", codes),
            "2FA 활성화 완료. 복구 코드 10개를 안전한 곳에 보관하세요."));
    }

    @Operation(summary = "2FA 검증 — login 흐름의 2단계")
    @SecurityRequirements()
    @PostMapping("/verify")
    public ResponseEntity<CommonResponse<TokenResponse>> verify(
        @Valid @RequestBody Verify2FARequest req, HttpServletRequest http
    ) {
        var device = http.getHeader("X-DEVICE-UUID");
        var ip = ClientIpUtil.resolveClientIp(http);
        var result = verify.handle(req.pendingToken(), req.code(), device, ip);
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            new TokenResponse(result.accessToken(), result.refreshToken(), "Bearer", 900L),
            "로그인 성공"));
    }
}

public record Enable2FARequest(@Min(0) @Max(999999) int code) {}
public record Verify2FARequest(@NotBlank String pendingToken, @Min(0) @Max(999999) int code) {}
```

---

## 7. SecretEncryption (AES-GCM)

```java
// infrastructure/security/crypto/SecretEncryption.java
@Component
public class SecretEncryption {

    private final SecretKey key;
    private final SecureRandom random = new SecureRandom();
    private static final int NONCE_LEN = 12;
    private static final int TAG_LEN_BIT = 128;

    public SecretEncryption(@Value("${app.secret-encryption.key}") String base64Key) {
        var raw = Base64.getDecoder().decode(base64Key);
        this.key = new SecretKeySpec(raw, "AES");
    }

    public byte[] encrypt(byte[] plain) {
        try {
            var nonce = new byte[NONCE_LEN];
            random.nextBytes(nonce);
            var c = Cipher.getInstance("AES/GCM/NoPadding");
            c.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(TAG_LEN_BIT, nonce));
            var ct = c.doFinal(plain);
            var out = ByteBuffer.allocate(nonce.length + ct.length);
            out.put(nonce).put(ct);
            return out.array();
        } catch (GeneralSecurityException e) { throw new IllegalStateException(e); }
    }

    public byte[] decrypt(byte[] envelope) {
        try {
            var nonce = Arrays.copyOfRange(envelope, 0, NONCE_LEN);
            var ct = Arrays.copyOfRange(envelope, NONCE_LEN, envelope.length);
            var c = Cipher.getInstance("AES/GCM/NoPadding");
            c.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(TAG_LEN_BIT, nonce));
            return c.doFinal(ct);
        } catch (GeneralSecurityException e) { throw new IllegalStateException(e); }
    }
}
```

```yaml
app:
  secret-encryption:
    key: ${SECRET_ENCRYPTION_KEY}     # base64 32-byte. KMS / Vault 주입.
```

---

## 8. 함정 모음

### 함정 1 — secret 평문 저장
DB 유출 = 모든 2FA 우회. **AES-GCM 암호화 + key rotation**.

### 함정 2 — recovery code 평문 저장
유출 시 모든 사용자 2FA 우회. **SHA-256 hash + 단일 사용**.

### 함정 3 — clock skew 미허용
서버 / 사용자 폰 시간 1초 차이만으로 거절. `windowSize=1` (±1).

### 함정 4 — brute force 미방어
6 digit = 100만. 무한 시도 = 분 단위. **5회 실패 = 15분 lock**.

### 함정 5 — 임시 토큰 (2fa_pending) 의 type 미구분
정식 access token 처럼 보호 endpoint 접근 가능. **`token_type=2FA_PENDING` claim** + 검증.

### 함정 6 — recovery code 생성 시 한 번에 노출 후 사라지지 않음
사용자가 잃어버리면 영구 우회 가능. **사용자가 회수 가능 (regenerate)** + 옛 codes 무효.

### 함정 7 — TOTP secret 을 QR 에만 노출하고 따로 안 보여줌
사용자가 다른 디바이스에 등록 못 함. **secret string 도 같이 노출 + 메모 권유**.

### 함정 8 — 2FA 비활성화 시 패스워드만 확인
세션 탈취된 공격자가 2FA 끔 = 무용지물. **패스워드 + TOTP 코드 둘 다** 요구.

### 함정 9 — TOTP 코드를 `String` 으로 받음
앞에 0 이 잘림 ("012345" → 12345). **String** 받고 정규식 검증, 또는 `int` 받고 zero-pad 처리.

### 함정 10 — recovery code 사용 시 알림 없음
도난 가능성 — 사용자에게 이메일/푸시 알림.

---

## 9. 운영 체크리스트

- [ ] AES-GCM secret encryption key 가 vault
- [ ] recovery codes hash 저장
- [ ] enable / disable / verify 실패 모두 audit log
- [ ] recovery 사용 시 사용자 알림
- [ ] secret encryption key rotation 정책
- [ ] 2FA 활성 사용자 비율 모니터링
- [ ] brute force lock count 모니터링 (이상 패턴 감지)

---

## 10. 관련

- [[login-jwt]] — 로그인 흐름과 통합
- [[email-verification]] · [[password-reset]] — 다른 토큰
- [[../common/security-config]] — JWT
- [[../pitfalls/transaction-pitfalls]] (예정)
- [[api-design|↑ api-design hub]]
