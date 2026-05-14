---
title: "PhoneNumber VO — 한국 휴대폰 + 정규화 + E.164"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:13:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - value-object
  - phone
---

# PhoneNumber VO — 한국 휴대폰 + 정규화 + E.164

**[[domain-model|↑ domain-model hub]]**  ·  ← [[value-objects]]

---

## 1. 코드

```java
// src/main/java/com/example/shop/domain/user/PhoneNumber.java
public record PhoneNumber(@JsonValue String value) {

    private static final Pattern KR_MOBILE = Pattern.compile(
        "^01[016789](-?[0-9]{3,4}){1}-?[0-9]{4}$"
    );

    public PhoneNumber {
        if (value == null)
            throw new IllegalArgumentException("phone required");
        if (!KR_MOBILE.matcher(value).matches())
            throw new IllegalArgumentException("invalid Korean mobile: " + value);
    }

    /** 정규화 — 하이픈 제거. "010-1234-5678" → "01012345678" */
    public String normalized() {
        return value.replaceAll("-", "");
    }

    /** E.164 국제 — "+82 10-1234-5678" → "+821012345678" */
    public String e164() {
        var stripped = normalized();
        return "+82" + stripped.substring(1);          // "01012345678" → "+821012345678"
    }

    /** 마스킹 — "010-1234-5678" → "010-1***-***8" */
    public String masked() {
        var n = normalized();
        if (n.length() != 11) return "***";
        return n.substring(0, 3) + "-" + n.charAt(3) + "***-***" + n.charAt(10);
    }
}
```

---

## 2. 왜 한국 휴대폰만

본 vault 는 한국 SaaS 가정.

```regex
^01[016789](-?[0-9]{3,4}){1}-?[0-9]{4}$
```

| 부분 | 의미 |
| --- | --- |
| `01[016789]` | 010 / 011 / 016 / 017 / 018 / 019 (3사 + 알뜰폰) |
| `(-?[0-9]{3,4})` | 가운데 자리 (구식 3 자리 또는 신식 4 자리) |
| `[0-9]{4}` | 끝 자리 |

매칭 예:
- ✅ `010-1234-5678` / `01012345678` / `011-123-4567` (옛 011)
- ❌ `02-1234-5678` (서울 지역번호) / `070-...` (인터넷 전화)

### 2.1 일반 전화번호 (집전화 / 사업장) 도 포함하려면

```regex
^(0(2|[3-6][1-5])-?[0-9]{3,4}-?[0-9]{4})|(01[016789]-?[0-9]{3,4}-?[0-9]{4})$
```

복잡해짐. 본 vault: 휴대폰만 (인증 / 알림 용도).

### 2.2 글로벌 확장 — E.164

해외 사용자 대상이면 — Google `libphonenumber` 사용:

```kotlin
implementation("com.googlei18n:libphonenumber:8.13.40")
```

```java
public PhoneNumber {
    var phoneUtil = PhoneNumberUtil.getInstance();
    try {
        var parsed = phoneUtil.parse(value, "KR");        // default region
        if (!phoneUtil.isValidNumber(parsed))
            throw new IllegalArgumentException("invalid phone");
    } catch (NumberParseException e) {
        throw new IllegalArgumentException("phone parse failed: " + e.getMessage());
    }
}
```

→ 본 vault 는 한국 단일 시장 가정 + 단순 regex. 글로벌 확장 시 libphonenumber.

---

## 3. 정규화 — 어떤 형식으로 저장

3 가지 옵션:

| 옵션 | 저장 형식 | 비고 |
| --- | --- | --- |
| **하이픈 제거** | `01012345678` (11 char) | **본 vault** — SMS / DB lookup 단순 |
| 하이픈 유지 | `010-1234-5678` (13 char) | UI 그대로 표시. 단 입력 변형 처리 부담 |
| E.164 | `+821012345678` (13 char) | 글로벌 표준 |

```java
// DB 저장 — 정규화된 형식
var phone = new PhoneNumber("010-1234-5678");
userJpaRepo.findByPhoneHash(HashUtil.sha256Hex(phone.normalized()));    // 검색 시 일관
```

> 💡 **왜 하이픈 제거?**
> 사용자 입력 다양 (`010-1234-5678` / `010 1234 5678` / `01012345678`) → 검색 시 mismatch.
> **저장은 정규화 (하이픈 X)**, **출력은 포맷팅 (010-1234-5678)** 권장.

### 3.1 출력 포맷팅 (옵션)

```java
public String formatted() {
    var n = normalized();
    if (n.length() == 11)  return n.substring(0, 3) + "-" + n.substring(3, 7) + "-" + n.substring(7);
    if (n.length() == 10)  return n.substring(0, 3) + "-" + n.substring(3, 6) + "-" + n.substring(6);
    return n;
}
```

UI 표시는 `formatted()`. 저장은 `normalized()`.

---

## 4. PII 보호

휴대폰은 PII (개인정보보호법). 처리 정책:

| 영역 | 정책 |
| --- | --- |
| DB 저장 | 평문 OK (UNIQUE 검색 필요) 또는 hash + 암호화 컬럼 (사고 시 PII 노출 방어) |
| 응답 (자기 정보) | OK (사용자 본인) |
| 응답 (다른 사용자) | 마스킹 (`010-1***-***8`) |
| 로그 | 마스킹 |
| 외부 SMS / AlimTalk 호출 | 평문 (호출에 필요) |

### 4.1 컬럼 암호화 + 해시 인덱스 — job-answer-be 식

```sql
ALTER TABLE users
    ADD COLUMN phone_encrypted BYTEA,        -- AES-GCM 암호화
    ADD COLUMN phone_hash CHAR(64);           -- SHA-256 (검색용)
CREATE UNIQUE INDEX ux_users_phone_hash ON users (phone_hash);
```

```java
// 저장
user.phoneEncrypted = secretEncryption.encrypt(phone.normalized().getBytes());
user.phoneHash = HashUtil.sha256Hex(phone.normalized());

// 검색
userRepo.findByPhoneHash(HashUtil.sha256Hex(input.normalized()));
```

→ DB 유출 시 평문 노출 X (decryption key 별도). search 는 hash 로.

본 vault 기본: **평문 (단순) + DB encryption at rest** (RDS encryption). 더 강한 보안 필요 시 컬럼 암호화.

---

## 5. DB 매핑

```sql
phone        VARCHAR(20),                    -- "01012345678" (정규화 후)
phone_hash   CHAR(64),                        -- 옵션 — 검색용 hash
```

```java
@Column(length = 20)
private String phone;                          // null 가능 (휴대폰 미제공 user)

@Column(name = "phone_hash", length = 64)
private String phoneHash;                      // 옵션
```

> 💡 VARCHAR(20) — 미래 글로벌 확장 대비 (E.164 13 char + 안전 마진).

---

## 6. equals / hashCode

```java
new PhoneNumber("010-1234-5678").equals(new PhoneNumber("01012345678"))
```

→ 위 코드는 **false** (값 다름). 문제?

**옵션 A — record 의 default (값 그대로)**: `false`
**옵션 B — normalized 기준**: equals override

```java
public record PhoneNumber(String value) {
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof PhoneNumber p)) return false;
        return normalized().equals(p.normalized());
    }
    @Override
    public int hashCode() { return normalized().hashCode(); }
}
```

본 vault: **옵션 B (normalized 기준)** — VO 의 "값 동등" 의미 살림.

단, record + override 는 약간 어색. 대안 — class 사용:
```java
public final class PhoneNumber {
    private final String value;
    // equals / hashCode override
}
```

또는 **항상 정규화된 값으로 생성**:
```java
public record PhoneNumber(String value) {
    public PhoneNumber {
        // 정규화 후 검증
        value = value.replaceAll("-", "");
        if (!NORMALIZED_PATTERN.matcher(value).matches()) ...
    }
}
```

→ 본 vault 옵션 — **정규화된 값을 record 의 value 로** 저장. record 의 default equals 가 자연스럽게 작동.

---

## 7. 함정 모음

### 함정 1 — 정규화 안 함
`"010-1234-5678"` 과 `"01012345678"` 가 다른 키로 저장 → 같은 사람이 두 user 가능.

### 함정 2 — UNIQUE 제약 없음
같은 휴대폰 2 user 가능. **`phone_hash` UNIQUE** 또는 `phone` UNIQUE (정규화 후).

### 함정 3 — equals 의 normalized 비교 누락
record 기본 equals 가 raw value 비교 → 같은 번호 다른 형식 = 다른 객체.

### 함정 4 — Bean Validation `@Pattern` 만 의존
DTO 단만 검증. 도메인 안에선 깨질 수 있음. **VO 도 검증**.

### 함정 5 — 응답에 raw 평문
다른 사용자의 휴대폰 노출 = PII 사고. **마스킹** (`010-1***-***8`).

### 함정 6 — SMS 호출 시 형식
NCP SENS 가 `+8210...` 또는 `01012345678` 또는 `010-1234-5678` 등 요구 형식 다름. **`normalized()` / `e164()` 메서드** 활용.

### 함정 7 — DB encryption 없음 + 평문
DB dump 유출 시 모든 사용자 휴대폰 노출. **RDS encryption at rest** 최소 + (선택) column-level 암호화.

### 함정 8 — 글로벌 확장 시 KR regex 그대로
해외 사용자 = 가입 실패. **libphonenumber + region 지정**.

### 함정 9 — VOIP / 060 등 비휴대폰
SMS 발송 불가. `01[016789]` regex 가 차단.

---

## 8. 관련

- [[domain-model|↑ domain-model hub]]
- [[value-objects]]
- [[email-vo]] — Email 의 정규화 (lowercase) 와 비교
- [[../phone-verification-impl]] — SMS 인증 흐름
- [[../security#3 민감정보]] — PII 처리 정책
