---
title: "SocialProviderType — 로그인 채널"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:38:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - enum
  - social-login
---

# SocialProviderType — 로그인 채널

**[[enums|↑ enums hub]]**  ·  관련: [[../../oauth2-social-login]]

> user 가 어떤 채널로 가입 / 로그인했는지. 동일 email 의 다른 provider 가입 충돌 처리에 핵심.

---

## 1. 값 정의

```java
// src/main/java/com/example/shop/domain/user/SocialProviderType.java
public enum SocialProviderType {

    /** email + password (자체 가입). 기본값. */
    LOCAL,

    /** Apple Sign In (iOS / macOS) */
    APPLE,

    /** Google OAuth 2.0 */
    GOOGLE,

    /** Kakao Login (한국) */
    KAKAO,

    /** Naver Login (한국) */
    NAVER;

    /** 외부 provider 인가 (LOCAL 만 false). */
    public boolean isExternal() { return this != LOCAL; }

    /** Apple 만 — appleSub 사용 필요. */
    public boolean usesAppleSub() { return this == APPLE; }

    /** 라벨 (UI 노출용). */
    public String label() {
        return switch (this) {
            case LOCAL -> "이메일";
            case APPLE -> "Apple";
            case GOOGLE -> "Google";
            case KAKAO -> "카카오";
            case NAVER -> "네이버";
        };
    }
}
```

---

## 2. 왜 이 5개

### 2.1 LOCAL — 자체 가입

당연. 모든 SaaS 의 기본.

### 2.2 KAKAO + NAVER — 한국 시장 필수

- **카카오** — 한국 3,500만 사용자. 가입율 ↑
- **네이버** — 카카오와 거의 동등. 둘 다 지원이 표준
- 한국 SaaS 가 카카오 / 네이버 안 지원 = 가입 friction 폭증

### 2.3 APPLE — iOS 앱 의무

- iOS 앱 App Store 정책: **소셜 로그인 제공 시 Apple Sign In 의무** (2020+)
- iOS 앱 없으면 옵션
- email 미제공 케이스 (첫 로그인만 제공) — 특수 처리 필요

### 2.4 GOOGLE — 글로벌 표준

- 한국 시장에선 카카오 / 네이버 보다 약하지만 글로벌·B2B 필수
- 회사 G Suite 사용자 / 개발자 / 해외 사용자

---

## 3. 더 필요한 후보 — 추가 안 한 것

### 후보: GITHUB / DISCORD / LINE 등

- 개발자 도구 (Stack Overflow / GitHub) → GITHUB 가치
- 게임 / 청소년 (Discord)
- 일본 (LINE)
- 해외 (Facebook / Twitter — 사용 ↓ 추세)

→ **사업 도메인** 에 따라 추가. enum 끝에 append (순서 의미 X).

### 후보: PHONE_ONLY (휴대폰만)

- 일부 한국 SaaS — 이메일 없이 휴대폰 번호 ID
- LOCAL 의 sub-type 으로 볼지 별도 type 으로 볼지 결정 필요
- **본 vault**: LOCAL 의 변형 (`User.email` nullable + `User.phone`)

---

## 4. provider 와 user 의 관계

### 4.1 같은 email 의 다른 provider — 정책 결정

| 정책 | 설명 | 본 vault |
| --- | --- | --- |
| **첫 가입 provider 만 허용** | LOCAL 로 가입 후 KAKAO 로 로그인 시도 = "이미 LOCAL 로 가입됨" | ✅ 기본 |
| **provider 분리 가입** | LOCAL `alice@x.com` + KAKAO `alice@x.com` 별도 user | 가능. complexity ↑ |
| **자동 병합** | 같은 email 자동 link | UX ↑ 단 보안 위험 (확인된 email 만) |

본 vault: **첫 provider 만**. 에러 메시지로 안내:

```
이미 카카오로 가입된 이메일입니다.
카카오로 로그인하시거나, 비밀번호로 로그인해 주세요.
```

### 4.2 DB

```sql
CREATE TABLE users (
    ...
    provider_type    VARCHAR(20) NOT NULL DEFAULT 'LOCAL',
    external_id      VARCHAR(100),                          -- provider 의 user ID (sub)
    apple_sub        VARCHAR(100),                          -- Apple 만 (refresh 검증용)
    password_hash    VARCHAR(255),                           -- LOCAL 만 NOT NULL
    ...
);

CREATE UNIQUE INDEX ux_users_provider_external
    ON users (provider_type, external_id)
    WHERE external_id IS NOT NULL;
```

```java
@Enumerated(EnumType.STRING)
@Column(name = "provider_type", nullable = false, length = 20)
private SocialProviderType providerType;
```

→ `(provider_type, external_id) UNIQUE` 로 provider 별 unique 보장.

---

## 5. 분기 로직 — 어디서

도메인 / Service 가 분기:

```java
// SocialLoginUseCase
if (provider == SocialProviderType.LOCAL)
    throw new IllegalArgumentException("LOCAL is not a social provider");

SocialAuthProvider authProvider = providers.of(provider);    // Strategy
SocialUserInfo info = authProvider.verify(externalToken);
...
```

[[../../oauth2-social-login#3.5]] 의 `SocialAuthProviderRegistry` 가 enum 으로 dispatch.

---

## 6. switch — exhaustive check

Java 21 sealed / switch:

```java
return switch (providerType) {
    case LOCAL -> "내 계정";
    case APPLE -> "Apple 로 로그인";
    case GOOGLE -> "Google 로 로그인";
    case KAKAO -> "카카오로 로그인";
    case NAVER -> "네이버로 로그인";
    // 새 값 추가 시 compile error → 모든 분기 강제 갱신
};
```

---

## 7. API 응답

```json
{
  "userId": "01HZ...",
  "email": "alice@x.com",
  "providerType": "KAKAO",
  "providerLabel": "카카오"             // 옵션
}
```

가입 / 로그인 API 의 응답에 providerType 노출. UI 가 "어떻게 가입됐는지" 표시.

---

## 8. 함정 모음

### 함정 1 — Apple 의 email 부재
첫 로그인 후엔 email 안 옴. **sub (appleSub) 기반 lookup** + 별도 컬럼.

### 함정 2 — Google 의 `email_verified=false`
드물게 가짜 email. 검증 필요.

### 함정 3 — externalId 의 type
Kakao 는 Long (`id: 123456`), Google 은 String (`sub: "abc..."`). **DB 는 String 통일**.

### 함정 4 — provider 추가 시 분기 누락
새 provider 추가 (GITHUB) — 모든 switch 분기 강제 갱신. **switch expression**.

### 함정 5 — LOCAL user 의 password_hash NOT NULL 제약
소셜 user 는 password 없음 — nullable. DB 마이그레이션:
```sql
ALTER TABLE users ALTER COLUMN password_hash DROP NOT NULL;
```

### 함정 6 — providerType 으로 user 분리
KAKAO 로 가입한 user 와 LOCAL 로 가입한 user 가 다른 사람일 수 있지만 같은 사람일 수도. **email 만으로 식별 X**.

### 함정 7 — 응답에 appleSub 노출
민감정보. 응답에 X. 내부 lookup 만.

---

## 9. 관련

- [[enums|↑ enums hub]]
- [[../../oauth2-social-login]] — provider 별 토큰 검증 구현
- [[../signup-impl]] — signup 시 providerType 분기
- [[../login-impl]] — login 시 providerType 일관성 검증
