---
title: "사용자 enumeration 방지 정책"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - security
  - enumeration
---

# 사용자 enumeration 방지 정책

**[[design-decisions|↑ design-decisions hub]]**

> "이메일 / 휴대폰이 가입돼 있는지 공격자가 추측할 수 있나" — 잘못 정책 = **가입자 list 유출** 또는 **UX 망함**.

---

## 1. 본 vault 결정

| SaaS 종류 | 정책 |
| --- | --- |
| 일반 B2C SaaS | **직관적** (가입 시 409, 비번 reset 시 "메일 발송 완료") |
| 금융 / 의료 | **차단** (모든 응답 동일 — "메일 확인하세요") |

---

## 2. Enumeration 의 위협

### 2.1 무엇

공격자가 응답 차이로 "이 이메일이 가입돼 있는지" 알아냄.

**시나리오 1: 가입 시도**
- 요청: `POST /signup { email: "victim@example.com", ... }`
- 응답 A: `409 "이미 가입된 이메일"` → ✅ 가입돼 있음 (정보 누출)
- 응답 B: `200 "메일 발송"` → 가입 안 됨

**시나리오 2: 비밀번호 재설정**
- 요청: `POST /password-reset { email: "victim@example.com" }`
- 응답 A: `404 "가입되지 않은 이메일"` → ❌ 가입 안 됨 (정보 누출)
- 응답 B: `200 "메일 발송"` → 가입돼 있음

### 2.2 왜 위험

- 공격자가 자동화 스크립트로 "어떤 이메일이 가입돼 있는지" 전수 조사 가능.
- 가입자 list = phishing / brute force 표적 list.
- 한국 SaaS 사고 사례: 다수 SaaS 의 가입자 list 가 dark web 에 거래.

---

## 3. 정책 결정 — 4구조

### 3.1 직관적 (본 vault default - 일반 SaaS)

**왜 적합**
- UX 우선 — 사용자가 "가입 안 됐어요" 즉시 안내 받음.
- 가입 거부 / 재설정 흐름이 명확.
- 한국 B2C SaaS 의 사실상 표준 (네이버 / 카카오 / 쿠팡 다 이 방식).

**왜 안 됨 (보안 critical)**
- 가입자 enumeration 가능 → list 유출 risk.

**대안과 왜 안 됨**
- 차단 정책 → UX 거침 (사용자가 "왜 메일 안 왔어요" 문의).

**보완 방어**
- Rate limit — 같은 IP 의 가입 / 재설정 시도 제한.
- CAPTCHA — 자동화 봇 방어.

---

### 3.2 차단 (금융 / 의료)

**왜 적합**
- 가입자 list 가 sensitive (금융 SaaS = 자금 보유 user).
- 법적 / 평판 risk ↑.

**왜 안 됨 (일반)**
- UX 거침 — 사용자가 가입 됐는지 안 됐는지 모름.

**구현**
- 모든 응답이 동일.
- 가입 시도 — 항상 "메일 확인하세요" + 가입 안 됐으면 메일 발송 안 함.
- 재설정 — 항상 "메일 발송 완료" + 가입 없으면 메일 발송 안 함.

**구체적 메시지**
```
POST /signup
→ 200 "회원가입을 시도했습니다. 인증 메일을 확인해주세요."

POST /password-reset
→ 200 "비밀번호 재설정 안내 메일을 발송했습니다."
```

→ 가입 / 미가입 모두 같은 메시지. 메일 받은 사람만 진행.

---

## 4. 비교

| 항목 | 직관적 | 차단 |
| --- | --- | --- |
| UX | ✅ 명확 | ⚠️ 헷갈림 |
| 보안 | ⚠️ enumeration 가능 | ✅ 차단 |
| 한국 SaaS 표준 | ✅ B2C | 금융 / 의료 |
| 구현 복잡도 | 단순 | 중간 (메일 발송 분기) |

---

## 5. 본 vault 구현 — 일반 SaaS

```java
@PostMapping("/auth/signup")
public ResponseEntity<?> signup(@Valid @RequestBody SignupRequest req) {
    try {
        userService.signup(req);
        return ResponseEntity.ok(...);
    } catch (EmailAlreadyExistsException e) {
        throw new BusinessException(ResponseCode.EMAIL_ALREADY_EXISTS);   // 409
    }
}
```

```java
@PostMapping("/auth/password-reset/request")
public ResponseEntity<?> requestPasswordReset(@RequestBody PasswordResetRequest req) {
    // 사용자가 있으면 메일 발송, 없으면 404
    try {
        passwordResetService.request(req.email());
        return ResponseEntity.ok(...);
    } catch (UserNotFoundException e) {
        throw new BusinessException(ResponseCode.USER_NOT_FOUND);          // 404
    }
}
```

---

## 6. 본 vault 구현 — 금융 / 의료 (차단)

```java
@PostMapping("/auth/signup")
public ResponseEntity<?> signup(@Valid @RequestBody SignupRequest req) {
    try {
        userService.signup(req);
    } catch (EmailAlreadyExistsException e) {
        // 무시 — 같은 응답
    }
    return ResponseEntity.ok(Map.of(
        "message", "회원가입을 시도했습니다. 메일을 확인해주세요."
    ));
}
```

```java
@PostMapping("/auth/password-reset/request")
public ResponseEntity<?> requestPasswordReset(@RequestBody PasswordResetRequest req) {
    try {
        passwordResetService.request(req.email());
    } catch (UserNotFoundException e) {
        // 무시 — 메일 발송 안 함, 응답은 동일
    }
    return ResponseEntity.ok(Map.of(
        "message", "비밀번호 재설정 안내 메일을 발송했습니다."
    ));
}
```

**왜 try/catch 후 무시**
- 응답 차이 없애기 — 가입 / 미가입 모두 같은 응답.
- 메일 발송 안 하면 — 가입 안 된 사용자는 메일 안 받음 (자연스럽게 enumeration 차단).

---

## 7. Timing Attack 방어

응답 시간 차이로도 enumeration 가능.

```
가입된 email: argon2 검증 200ms
가입 안 된 email: DB 조회만 5ms (응답 빠름)
```

→ 응답 시간 측정으로 user 존재 추측.

**방어**
```java
public boolean login(String email, String password) {
    var user = users.findByEmail(email).orElse(null);

    // user 없어도 dummy hash 검증 (같은 시간 소비)
    if (user == null) {
        encoder.matches(password, DUMMY_HASH);   // ~200ms
        return false;
    }
    return encoder.matches(password, user.passwordHash().value());
}
```

**왜**
- "user 없음" 응답 5ms vs "비밀번호 틀림" 응답 200ms = timing 차이로 enumeration.
- dummy hash 검증으로 시간 균일화.

---

## 8. Rate Limit (필수 보완)

직관적 정책 + rate limit = enumeration 비용 ↑.

```
IP 당:
- 가입 시도: 1시간 5회
- 비번 재설정: 1시간 3회
- 로그인: 1분 5회

User 당 (login):
- 5회 실패 → 30분 lock
```

**왜**
- enumeration 자동화 스크립트가 분당 100+ 시도 → IP block.
- 사용자 friction ↓ (정상 사용자는 영향 X).

자세히: [[../rate-limiting]].

---

## 9. CAPTCHA (옵션)

- 가입 / 비번 재설정 시 reCAPTCHA / hCaptcha.
- 자동화 봇 방어.

**언제 적합**
- 어뷰즈 폭증 시 — 활성화.
- 일반 시기에 — UX 부담.

**Trade-off**
- UX 부담 vs 봇 방어.
- 본 vault: 처음엔 X, 필요 시 활성화.

---

## 10. 함정 모음

### 함정 1 — 응답 메시지로 enumeration
"이미 가입된 이메일" + "가입되지 않은 이메일" = 분명한 신호.
→ 차단 정책 또는 rate limit + CAPTCHA.

### 함정 2 — HTTP status code 로 enumeration
409 (이미 가입) vs 200 (메일 발송) = 분명한 신호.
→ 차단 정책 시 status 도 통일.

### 함정 3 — 응답 시간 차이
DB 조회 5ms vs hash 검증 200ms.
→ dummy hash 검증으로 균일화.

### 함정 4 — Rate limit 없음
enumeration 자동화 무제한.
→ IP + user 별 limit.

### 함정 5 — 비번 재설정 시 메일 발송 여부로 enumeration
"메일 발송 완료" 응답 후 실제 메일 안 옴 → 가입 안 됨.
→ 차단 정책 + 모든 응답 통일.

### 함정 6 — 가입 시 이메일 / 휴대폰 둘 다 검증
이메일 가입 후 휴대폰 가입 시도 → "이미 가입된 휴대폰" 응답으로 휴대폰 enumeration.
→ 차단 정책 시 둘 다 동일.

### 함정 7 — 직관적 정책 + CAPTCHA 없음
어뷰즈 폭증 시 무방비.
→ rate limit + CAPTCHA fallback.

### 함정 8 — 로그인 실패 메시지 분리
"비밀번호 틀림" vs "user 없음" → enumeration.
→ "이메일 또는 비밀번호가 일치하지 않습니다" 통일.

### 함정 9 — 비번 재설정 후 다른 응답
"가입 안 된 사용자에게도 메일 발송 완료" 라고 응답 후 사용자가 메일 안 받음 → 의심.
→ 사용자에게 안내 모달 ("메일이 안 오면 가입 안 된 이메일일 수 있어요" — UX 명시).

### 함정 10 — 비번 재설정 link 의 user_id 노출
URL 에 user_id 가 있으면 → 가입 user_id list 누출.
→ token 만 사용 (token 의 hash 로 user 찾음).

---

## 11. 다른 컨텍스트

### 11.1 일반 B2C

```yaml
enumeration: explicit
rate-limit: enabled (5/h signup, 3/h reset)
captcha: optional (어뷰즈 시 활성)
```

### 11.2 금융 / 의료

```yaml
enumeration: obscured
rate-limit: strict
captcha: always
identity-verification: pass-required
```

### 11.3 B2B SaaS

```yaml
enumeration: explicit
reason: 기업 가입자 list 보호 우선순위 ↓
```

---

## 12. 관련

- [[design-decisions|↑ hub]]
- [[../security]] — auth 보안 전반
- [[../rate-limiting]] — IP / user rate limit
- [[../signup-impl]] · [[../login-impl]] · [[../password-reset-impl]]
- 외부 — OWASP Authentication Cheat Sheet, NIST 800-63B
