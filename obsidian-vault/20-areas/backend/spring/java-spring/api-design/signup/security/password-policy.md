---
title: "패스워드 정책 — NIST 800-63B + pwned check"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - password-policy
  - nist
---

# 패스워드 정책 — NIST 800-63B + pwned check

**[[security|↑ security hub]]**

> "어떤 비밀번호를 받을 것인가" — 옛 정책 (대소문자+특수문자 강제) 은 오히려 보안 ↓. NIST 2017+ 의 새 정책.

---

## 1. 본 vault 결정

```yaml
password-policy:
  min-length: 8
  max-length: 128
  complexity-required: false              # 길이만, 문자 종류 강제 X
  contain-spaces: false                   # 공백 차단
  pwned-check: true                       # haveibeenpwned API
  similar-to-username: forbidden           # email / name 과 유사 차단
```

---

## 2. NIST 800-63B (2017+) 의 핵심 변경

### 2.1 옛 정책의 문제

**옛 정책 (2010년대)**
- 대소문자 + 숫자 + 특수문자 강제 (`Password1!`).
- 90일마다 강제 변경.
- 8자 짧으면 OK.

**문제**
- 사용자가 패턴화 (`Password1!` → `Password2!` → `Password3!`).
- 짧은 + 추측 가능 → brute force 쉬움.
- 잦은 변경 = 사용자 메모지 / 1Password 미설치 시 보안 ↓.

### 2.2 NIST 새 정책

| 항목 | 새 정책 | 옛 정책 |
| --- | --- | --- |
| 최소 길이 | 8자 (길수록 더 좋음) | 8자 |
| 최대 길이 | **128+** (자르지 말 것) | 보통 16-20 |
| 복잡도 강제 | **X** (사용자 자유) | 대소+숫자+특수 |
| 강제 변경 | **X** | 90일 |
| 알려진 유출 차단 | **권장** | X |
| 사용자명 포함 차단 | 권장 | X |

**왜 복잡도 강제 X**
- "Password1!" 가 "supercalifragilisticexpialidocious" 보다 약함 (entropy ↓).
- 복잡도 강제 = 사용자가 패턴화 (사실상 entropy ↓).
- 길이 우선 = entropy ↑.

**왜 강제 변경 X**
- 잦은 변경 = 메모지 / 약한 변형 (Password1 → Password2).
- 유출 발견 시만 강제.

---

## 3. 길이 정책 — 8~128

### 3.1 최소 8

**왜**
- 8자 = 약 60 bit entropy (영숫자 가정).
- argon2id 검증 + 8자 = brute force 불가.
- 너무 짧으면 (4-6) → 100만 경우 미만.

**왜 더 길게 (12+) 권장 X**
- 너무 강제 = 가입 friction.
- 8 + pwned check + lock = 충분.

### 3.2 최대 128

**왜**
- passphrase ("correct horse battery staple") 권장 — 길이 25+.
- argon2id 의 입력 한계 (사실상 무한).
- 너무 짧음 (32) 강제 = 사용자 제약.

**왜 1024 / 무한 X**
- DoS 위험 — 길이 ↑ → argon2 검증 시간 ↑.
- 128 = 산업 표준 균형.

---

## 4. 공백 / 특수문자 정책

### 4.1 공백 차단

```java
if (plain.contains(" ")) throw ...
```

**왜**
- 사용자 실수 (앞 / 뒤 공백 추가) → 다음 로그인 실패.
- UX 헷갈림 ("내 비번 맞는데 왜?").

**대안**
- trim → 옛 공백 의도 있는 사용자에게는 보안 약화.
- 본 vault: 차단 (단순 + 일관성).

### 4.2 모든 ASCII / Unicode OK

```java
// NIST 권장 — 모든 인쇄 가능 문자 허용
```

**왜**
- passphrase 의 단어 사이 공백 (`correct horse battery`) — 단 본 vault 는 차단.
- 한국어 비밀번호 (`안녕하세요123`) — 권장은 OK.

---

## 5. pwned check (haveibeenpwned API)

### 5.1 무엇

- 5억+ 유출된 비밀번호 DB.
- API: `https://api.pwnedpasswords.com/range/<sha1[0:5]>`
- k-anonymity — 비밀번호 자체 안 보냄.

### 5.2 흐름

```
1. 사용자 입력: "password123"
2. SHA-1: A94A8FE5CCB19BA61C4C0873D391E987982FBBD3
3. API 호출: GET /range/A94A8     (앞 5자만)
4. 응답: [
     "FE5CCB19BA61C4C0873D391E987982FBBD3:1234567",  ← 1234567 회 유출
     "FE5CDB19BA61C4C0873D391E987982FBBD3:1",
     ...
   ]
5. 우리 hash 의 뒤 35자와 매칭 → 발견 = pwned
```

### 5.3 구현

```java
@Component
public class PwnedPasswordChecker {
    private final WebClient webClient = WebClient.create("https://api.pwnedpasswords.com");

    public boolean isPwned(String plain) {
        String sha1 = sha1Hex(plain).toUpperCase();
        String prefix = sha1.substring(0, 5);
        String suffix = sha1.substring(5);

        String response = webClient.get()
            .uri("/range/{prefix}", prefix)
            .retrieve()
            .bodyToMono(String.class)
            .block();

        return Arrays.stream(response.split("\n"))
            .map(line -> line.split(":"))
            .anyMatch(parts -> parts[0].equalsIgnoreCase(suffix));
    }

    private String sha1Hex(String input) {
        try {
            var digest = MessageDigest.getInstance("SHA-1");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) { throw new RuntimeException(e); }
    }
}
```

### 5.4 왜 k-anonymity

- 우리가 사용자 비밀번호를 외부에 안 보냄.
- 앞 5자 (SHA-1) 만 전송 = 약 1000개 후보 중 1개 = 누가 어떤 비밀번호인지 추측 불가.

### 5.5 함정

- API rate limit — 분당 수만 요청 가능 (충분).
- 응답 길이 — 가끔 100KB+ (5억 비밀번호 prefix 분포).
- 외부 의존 — 장애 시 fallback (warn but allow).

---

## 6. 사용자명 / 이메일 포함 차단

```java
if (plain.toLowerCase().contains(email.localPart().toLowerCase())) throw ...
if (plain.toLowerCase().contains(name.toLowerCase())) throw ...
```

**왜**
- "alice123" / "alice2024" → 추측 쉬움.
- 사용자 정보 유출 시 다른 사이트 비밀번호로 시도 가능.

**한계**
- 부분 매칭 → 완벽 X.
- 한국 이름 (3자) 와 우연 일치 가능.

---

## 7. PasswordPolicy 구현

```java
@Component
@RequiredArgsConstructor
public class PasswordPolicy {

    private final PwnedPasswordChecker pwnedChecker;

    public void validate(String plain, String email, String name) {
        if (plain == null)
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT, "password required");
        if (plain.length() < 8 || plain.length() > 128)
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password length must be 8-128");
        if (plain.contains(" "))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password cannot contain spaces");
        if (email != null && containsIgnoreCase(plain, localPart(email)))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password cannot contain email");
        if (name != null && containsIgnoreCase(plain, name))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password cannot contain name");
        if (pwnedChecker.isPwned(plain))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "password found in known breaches; choose another");
    }
}
```

**왜 BusinessException (도메인 검증 후)**
- Bean Validation `@Size` / `@Pattern` 으로 일부 가능하지만 pwned check 는 외부 호출.
- 도메인 service 에서 검증 = 일관성.

---

## 8. 비밀번호 변경 정책

### 8.1 옛 비밀번호 입력 강제

```java
public void changePassword(UserId userId, String oldPlain, String newPlain) {
    var user = users.findById(userId).orElseThrow();
    if (!encoder.matches(oldPlain, user.passwordHash().value()))
        throw new InvalidCredentialsException();
    passwordPolicy.validate(newPlain, user.email().value(), user.name());
    user.changePassword(new PasswordHash(encoder.encode(newPlain)));
    refreshTokens.revokeAllForUser(userId, "PASSWORD_CHANGED");
}
```

**왜 옛 비밀번호 검증**
- 도난 access token 으로 비밀번호 변경 시도 차단.
- 본인 확인.

### 8.2 같은 비밀번호 차단

```java
if (encoder.matches(newPlain, user.passwordHash().value()))
    throw new BusinessException(..., "new password same as old");
```

**왜**
- 패스워드 회전 정책 (선택) 시 효과.
- "변경했는데 같음" 의도 불명.

### 8.3 옛 비밀번호 N개 차단 (선택)

- 옛 비밀번호 5개 hash 별도 저장 → 재사용 차단.
- 보안 강한 도메인만.

---

## 9. 함정 모음

### 함정 1 — 복잡도 강제 (대소문자 + 특수)
사용자 패턴화 → 보안 ↓.
→ 길이만 강제.

### 함정 2 — 짧은 max length (16-20)
passphrase 제한.
→ 128.

### 함정 3 — 90일 강제 변경
약한 변형 (Password1 → 2).
→ 유출 발견 시만 강제.

### 함정 4 — pwned check 없음
"password123" 통과 → 즉시 brute force.
→ haveibeenpwned API.

### 함정 5 — 사용자명 / 이메일 차단 없음
"alice2024" / "alice@example.com" 추측 쉬움.
→ 차단.

### 함정 6 — 공백 차단 없음 (trim 정책)
사용자 실수로 공백 추가 → 다음 로그인 실패.
→ 차단 (단순).

### 함정 7 — bcrypt 72-byte 한계
긴 passphrase silent truncate.
→ argon2id (제한 없음).

### 함정 8 — 비밀번호 변경 시 옛 비밀번호 검증 X
도난 access token 으로 비밀번호 변경 = 영구 탈취.
→ 옛 비밀번호 검증 필수.

### 함정 9 — 변경 후 refresh revoke 누락
공격자가 옛 refresh 로 계속 access 갱신.
→ revokeAllForUser.

### 함정 10 — 비밀번호 입력에 자동완성 권장
브라우저 / 매니저 자동완성 안 됨 정책 (autocomplete="off") → 사용자가 약한 비밀번호 선택.
→ autocomplete="new-password" / "current-password" 명시.

### 함정 11 — 응답 시 비밀번호 강도 정보 노출
"이 비밀번호는 너무 약합니다" 메시지에 너무 자세한 정보.
→ 단순 "더 강한 비밀번호" 안내.

### 함정 12 — pwned check 외부 의존 — 장애 시 무방비
API 다운 시 모두 통과.
→ fallback (warn + allow), 모니터링.

---

## 10. 관련

- [[security|↑ security hub]]
- [[../design-decisions/password-hash]] — 알고리즘 선정 (Argon2id)
- [[../domain-model/password-hash-vo]] — PasswordHash VO
- [[../signup-impl]] · [[../password-reset-impl]] — 구현
- 외부 — NIST SP 800-63B (2017+), haveibeenpwned k-anonymity
