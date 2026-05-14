---
title: "2FA 정책 — 도입 시점 + 방식"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - 2fa
  - totp
---

# 2FA 정책 — 도입 시점 + 방식

**[[design-decisions|↑ design-decisions hub]]**

> "2FA 를 어떤 사용자에게 어떻게 강제하나" — 무차별 강제는 UX 망함, 안 함은 보안 위험.

---

## 1. 본 vault 결정

- **일반 user**: optional (설정에서 활성).
- **민감 작업** (비밀번호 변경 / 결제 / 인출): step-up auth (해당 작업 시 2FA 입력).
- **ADMIN role**: 가입 시 강제 활성.

---

## 2. 시점별 정책 — 4구조

### 2.1 가입 시 강제

**왜 적합한 케이스**
- 금융 / 의료 — 법적 요구 또는 사고 시 책임 ↑.
- 매우 민감한 데이터 (의료 기록, 송금).

**왜 안 됨 (일반 SaaS)**
- 가입 friction 폭증 — 회원가입 이탈률 30%+ 증가 (산업 평균).
- 일반 사용자가 2FA app 설치 부담.

**대안**
- Optional + step-up — UX 와 보안 균형.

---

### 2.2 Optional (설정에서 활성)

**왜 적합 (본 vault default)**
- 의지 있는 사용자만 활성 → UX 영향 X.
- 침해 발생 시 사용자 책임 명확.

**한계**
- 활성률 낮음 (~5%).
- 강한 보안 필요한 작업 보호 X.

**보완**
- Step-up auth 같이 적용.

---

### 2.3 Step-up (민감 작업 시 강제)

**왜 필수 (보안 critical)**
- 모든 작업에 2FA 강제 X → 일부 작업만 2FA 입력 (UX 균형).
- 비밀번호 변경 / 결제 / 인출 / API key 생성 / OAuth grant 같은 작업.

**구현**
```
1. 사용자가 "비밀번호 변경" 시도
2. 서버: "step_up_required" 응답 (2FA token 요구)
3. 클라: 2FA 코드 입력 화면
4. 서버: 2FA 검증 → step_up_token 발급 (TTL 5분)
5. 비밀번호 변경 API 호출 시 step_up_token 포함
6. 서버: 검증 + 비밀번호 변경
```

**왜 step_up_token (직접 검증 X)**
- 사용자 흐름 — UX 끊김 ↓.
- step_up_token 의 짧은 TTL = 도난 영향 시간 ↓.

---

### 2.4 관리자 (ADMIN) 강제

**왜 필수**
- 권한 ↑ → 사고 시 영향 ↑.
- ADMIN 계정 탈취 = 전체 user 영향.

**구현**
```java
public class User {
    public void promoteToAdmin() {
        ensureTotpEnabled();         // 2FA 안 됐으면 reject
        this.role = Role.ADMIN;
    }
}
```

---

## 3. 2FA 방식 비교

### 3.1 TOTP (Time-based One-Time Password)

**왜 default (본 vault)**
- 표준 (RFC 6238).
- 무료 — Google Authenticator / Microsoft Authenticator / 1Password.
- 오프라인 작동 (SMS 같은 외부 의존 X).
- 한 번 설정 후 유지.

**한계**
- 사용자가 앱 설치 / 설정 부담.
- 단말 분실 시 recovery code 필요.

---

### 3.2 SMS

**왜 적합**
- 별도 앱 X — 가입 흐름 짧음.
- 한국 휴대폰 보유율 99%.

**왜 안 됨 (보안)**
- SIM swap 공격 (휴대폰 번호 탈취).
- 통신사 의존 (delivery 실패 가능).
- NIST 2017 ~ SMS-based 2FA 비추.

**언제 적합**
- TOTP fallback (앱 분실 시).

---

### 3.3 WebAuthn / FIDO2 (Passkey)

**왜 미래**
- 가장 강한 보안 — phishing 저항.
- Apple / Google 표준 (Passkey).
- iOS 16+ / Android 14+ 지원.

**왜 아직 default 아님**
- 채택률 낮음 (사용자 학습 부담).
- 라이브러리 / Spring 통합이 신생.

**언제 도입**
- 보안 critical SaaS — 2026~ 흔해질 예정.

---

### 3.4 Email OTP

**왜 적합**
- 별도 앱 / 인프라 X.
- 이메일 이미 있음 (가입 시).

**왜 안 됨**
- Email 자체가 탈취되면 무용지물.
- 발송 지연 (~분).

**언제 적합**
- 다른 2FA 다 안 되는 fallback.

---

## 4. TOTP 구현

```kotlin
// build.gradle
implementation("com.warrenstrange:googleauth:1.5.0")
implementation("com.google.zxing:core:3.5.3")              // QR code
```

```java
@Service
public class TotpService {
    private final GoogleAuthenticator gAuth = new GoogleAuthenticator();

    public TotpSetup setup(UserId userId) {
        var key = gAuth.createCredentials();                // random 32-byte secret
        var qrUrl = GoogleAuthenticatorQRGenerator.getOtpAuthURL(
            "Yule Studio", userId.value(), key);

        // DB 에 secret 임시 저장 (PENDING - 활성화 전)
        totpRepo.save(new TotpEntity(userId, key.getKey(), false));

        return new TotpSetup(qrUrl, key.getKey());
    }

    public boolean verify(UserId userId, int code) {
        var totp = totpRepo.findByUserId(userId).orElseThrow();
        return gAuth.authorize(totp.secret(), code);
    }

    public void activate(UserId userId, int code) {
        if (!verify(userId, code)) throw new InvalidTotpException();
        totpRepo.markActive(userId);                         // PENDING → ACTIVE
    }
}
```

**왜 PENDING / ACTIVE 분리**
- 사용자가 secret 받고 첫 verify 통과해야 active.
- secret 받았는데 잊어버린 사용자 → PENDING row 만 남음 → 비활성으로 취급.

---

## 5. Recovery Code

```
가입 시 2FA 활성 →
서버: 10 개 recovery code 발급 (8자리 random)
사용자: 안전한 곳에 보관 (1password / 종이)

단말 분실 시:
사용자 입력: recovery_code
서버: hash 매칭 → 새 TOTP 설정 가능 / 임시 로그인 통과
사용된 recovery_code 는 invalidate
```

**왜 필수**
- 단말 분실 = TOTP secret 손실 = 영구 잠금.
- recovery code 가 emergency exit.

**저장**
- DB 에 hash (`recovery_codes` 테이블, `code_hash CHAR(64), status: ACTIVE/USED`).
- 평문 X — 도난 시 즉시 사용 가능.

---

## 6. Step-up Auth 구현

```java
@PostMapping("/me/password")
public ResponseEntity<?> changePassword(
    @RequestBody ChangePasswordRequest req,
    @RequestHeader("X-Step-Up-Token") String stepUpToken
) {
    stepUpAuthService.verify(stepUpToken, "password_change");
    // 검증 통과 시 비밀번호 변경
    userService.changePassword(authUser, req.oldPassword(), req.newPassword());
    return ResponseEntity.ok();
}
```

```java
@PostMapping("/me/step-up")
public StepUpResponse stepUp(
    @RequestBody StepUpRequest req,                    // totpCode
    @AuthenticationPrincipal AuthUser authUser
) {
    if (!totpService.verify(authUser.id(), req.totpCode())) {
        throw new InvalidTotpException();
    }
    var token = stepUpAuthService.issue(authUser.id(), req.action());
    return new StepUpResponse(token);                  // TTL 5m
}
```

**왜 step_up_token**
- 검증 결과를 짧은 시간 (5m) 동안 reusable.
- "비밀번호 변경" 흐름의 여러 단계에서 재사용 (확인 화면 → 실제 변경).

---

## 7. 함정 모음

### 함정 1 — 가입 시 2FA 강제 (일반 SaaS)
이탈률 30%+ 증가.
→ Optional + step-up.

### 함정 2 — 2FA 활성 후 recovery code 없음
단말 분실 = 영구 잠금.
→ 활성 시 10 개 recovery code 발급 + 보관 안내.

### 함정 3 — TOTP secret 평문 저장
DB 유출 = 모든 2FA 우회.
→ 암호화 또는 별도 vault.

### 함정 4 — SMS 만 2FA 사용
SIM swap 공격에 취약.
→ TOTP 우선, SMS 는 fallback.

### 함정 5 — 모든 작업에 2FA 강제
UX 망함.
→ 민감 작업만 step-up.

### 함정 6 — 2FA 코드 검증 시 시간 윈도우 너무 김 (5분+)
도난 시 사용 가능 시간 ↑.
→ ±30초 (Google Authenticator 표준).

### 함정 7 — Step-up token TTL 너무 김 (1h+)
도난 시 1시간 동안 민감 작업 가능.
→ 5분.

### 함정 8 — 2FA 비활성화 시 step-up 없음
공격자가 2FA 비활성 → 보호 우회.
→ 2FA 끄는 작업도 step-up 필요.

### 함정 9 — Recovery code 단일 사용 안 함
한 code 무한 사용 → 도난 시 영구 우회.
→ status=USED 전이 + 모든 사용된 code log.

### 함정 10 — TOTP secret 의 QR 화면 cache
브라우저 / 스크린샷 유출 → secret 노출.
→ QR 화면에 "스크린샷 X" 경고 + 짧은 표시 시간.

### 함정 11 — 같은 TOTP 코드 재사용
사용자가 같은 30초 윈도우에 재로그인 시도 → replay attack 가능.
→ 사용된 code 의 마지막 timestamp 기록.

### 함정 12 — ADMIN 2FA 미강제
admin 계정 탈취 = 전체 사고.
→ ADMIN 강제.

---

## 8. 다른 컨텍스트

### 8.1 금융 / 의료

```yaml
two-factor-auth:
  enforcement: required-at-signup
  method: totp + pass-identity
  reason: 법적 요구
```

### 8.2 일반 B2C SaaS

```yaml
two-factor-auth:
  enforcement: optional + step-up
  method: totp
  admin: required
```

### 8.3 미래 (Passkey)

```yaml
two-factor-auth:
  primary: passkey (webauthn)
  fallback: totp
  reason: phishing 저항
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[password-hash]] — 비밀번호 변경 시 step-up
- [[../two-factor-auth]] — 자세한 구현
- [[../security]] — step-up auth 전반
- 외부 — RFC 6238 TOTP, NIST SP 800-63B, WebAuthn spec
