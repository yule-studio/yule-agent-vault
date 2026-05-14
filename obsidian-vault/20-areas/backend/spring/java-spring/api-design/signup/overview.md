---
title: "auth §1 — 전체 흐름 overview"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - overview
---

# auth §1 — 전체 흐름 overview

**[[signup|↑ hub]]**  ·  → [[prerequisites]]

> 회원가입부터 패스워드 리셋까지의 **end-to-end auth 흐름** 을 한 페이지에 그림.

---

## 1. 사용자 여정 (User Journey)

```mermaid
flowchart TD
    Visit[첫 방문] --> SignupForm[회원가입 폼]
    SignupForm -->|이메일·패스워드·약관| Signup["POST /auth/signup"]
    Signup --> Pending["user.status = PENDING_VERIFICATION"]
    Pending --> Event[UserRegistered event → outbox]
    Event --> SendMail[이메일 발송 별도 워커]
    SendMail --> MailBox[메일 박스]
    MailBox -->|인증 링크 클릭| EmailConfirm["POST /auth/verify/email/confirm"]
    EmailConfirm --> Active["user.status = ACTIVE"]

    Active --> LoginForm[로그인 폼]
    LoginForm -->|이메일·패스워드| Login["POST /auth/login"]
    Login --> Tokens["access 15m + refresh 14d rotation"]
    Tokens --> Use[정상 사용]

    Use -->|access 만료| Refresh["POST /auth/token/refresh"]
    Refresh --> NewTokens[new access + new refresh rotation]
    NewTokens --> Use

    Use -->|로그아웃| Logout["POST /auth/logout"]
    Logout --> RevokeRT[refresh REVOKED]

    Use -.->|패스워드 분실| ResetReq["POST /auth/password-reset/request"]
    ResetReq --> MailSms[메일 또는 SMS]
    MailSms -->|토큰 클릭| ResetConfirm["POST /auth/password-reset/confirm"]
    ResetConfirm --> Reset[passwordHash 갱신 + 모든 refresh 무효]

    style Active fill:#dbeafe
    style Pending fill:#fef3c7
    style Reset fill:#fecaca
```

---

## 2. 휴대폰 인증이 포함된 흐름 (한국 표준)

한국 SaaS 는 보통 **휴대폰 인증을 가입 전제** 로 함 (SMS / AlimTalk).

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Web as 가입 폼
    participant API as Backend
    participant Redis
    participant SMS as NCP SENS / AlimTalk

    User->>Web: 휴대폰 번호 입력
    Web->>API: POST /auth/verify/phone/request<br/>{ phone: "010-0000-0000" }
    API->>SMS: 6자리 코드 발송
    SMS-->>User: 인증번호 SMS

    User->>Web: 6자리 코드 입력
    Web->>API: POST /auth/verify/phone/confirm<br/>{ phone, code }
    API->>Redis: verifiedAt 마킹 (TTL 10분)
    API-->>Web: phoneAuthToken 발급

    User->>Web: 회원가입 폼 작성
    Web->>API: POST /auth/signup<br/>{ ..., phone, phoneAuthToken }
    API->>Redis: phoneAuthToken 검증 (10분 이내?)
    Redis-->>API: OK
    API->>API: user 생성 + phone 저장
    API-->>Web: 가입 완료
```

상세: [[phone-verification-impl]].

---

## 3. 도메인 / 인프라 의존성 그림

```mermaid
flowchart TD
    Security["Spring Security<br/>+ JwtAuthFilter<br/>(모든 endpoint 가 통과)"]
    Security --> SignupCtl[SignupCtl]
    Security --> LoginCtl[LoginCtl]
    Security --> MeCtl[MeCtl]

    SignupCtl --> SignupUC["SignupUC<br/>@Transactional"]
    LoginCtl --> LoginUC["LoginUC<br/>@Transactional"]
    MeCtl --> UserQ[User Query Service]

    SignupUC --> Domain
    LoginUC --> Domain
    UserQ --> Domain

    Domain["Domain<br/>User Aggregate / Email / PasswordHash / VO"]
    Domain --> UserRepo[UserRepository port]
    Domain --> RtRepo[RefreshTokenRepository port]

    UserRepo -.->|implements| JpaUser[JpaUserAdapter]
    RtRepo -.->|implements| JpaRt[JpaRefreshAdapter]

    JpaUser --> DB[(PostgreSQL)]
    JpaRt --> DB

    SignupUC -.-> SES[AWS SES / SendGrid<br/>이메일]
    SignupUC -.-> NCP[NCP SENS / AlimTalk<br/>SMS]
    LoginUC -.-> Social[Apple / Google / Kakao API<br/>소셜]

    style Domain fill:#fef3c7
    style DB fill:#fecaca
    style Security fill:#dbeafe
```

> 💡 **점선** = 외부 / implements 관계. **실선** = 직접 호출.

---

## 4. 어떤 endpoint 가 비인증인가

| Endpoint | 인증 | 비고 |
| --- | --- | --- |
| `POST /auth/signup` | ❌ | 회원가입 자체 |
| `POST /auth/signup/social` | ❌ | provider 토큰만 검증 |
| `POST /auth/verify/email/request` | ✅ 인증 | 로그인 후 본인 메일 |
| `POST /auth/verify/email/confirm` | ❌ | 메일의 토큰 검증 |
| `POST /auth/verify/phone/request` | ❌ | 가입 전 + 가입 후 (재인증) |
| `POST /auth/verify/phone/confirm` | ❌ | 위와 같이 |
| `POST /auth/login` | ❌ | 로그인 |
| `POST /auth/login/social` | ❌ | 소셜 로그인 |
| `POST /auth/token/refresh` | ❌ | refresh token 만 검증 (Body) |
| `POST /auth/logout` | ❌ | refresh token 만 |
| `POST /auth/password-reset/request` | ❌ | 비인증 (잊었으니까) |
| `POST /auth/password-reset/confirm` | ❌ | 토큰 검증만 |
| `GET /me` | ✅ | access token 필수 |

→ SecurityConfig 의 `permitAll` 매트릭스. [[security#3 endpoint 인가 매트릭스]] 참조.

---

## 5. 각 흐름의 핵심 결정점

| 단계 | 핵심 결정 | 영향 |
| --- | --- | --- |
| 가입 — 인증 정책 | 이메일 / 휴대폰 / 둘 다 | 가입 friction vs 보안 |
| 가입 — 자동 로그인 | Yes / No | 본 vault: No (인증 후 별도 로그인) |
| 로그인 — 토큰 모델 | JWT only / Session / 둘 다 | 본 vault: access JWT + refresh opaque |
| 로그인 — refresh 저장 | RDB / Redis | 본 vault: RDB (영속) + 점진 Redis |
| 토큰 갱신 — rotation | Yes / No | 본 vault: Yes (탈취 감지) |
| 패스워드 리셋 — 채널 | 이메일 / SMS | 보통 이메일 (가입 인증 동일) |
| 다중 디바이스 — 동시 로그인 | 허용 / 단일 | 본 vault: 허용 (RefreshToken row N개) |

각 결정의 trade-off + 권장은 [[design-decisions]].

---

## 6. 이 폴더 코드의 재사용성

이 폴더의 코드는 **2 부분으로 구성**:

### A. 도메인 / 표준 부분 (그대로 복사)

- Value Objects (Email, PasswordHash, PhoneNumber, UserId, ...)
- User Aggregate + 상태 머신
- Domain Events
- Repository port + JPA Adapter
- Standard responses ([[../../common/response-envelope]])
- SecurityConfig + JwtTokenProvider ([[../../common/security-config]])

→ **거의 모든 SaaS 에서 그대로 사용 가능**.

### B. 정책 / 비즈니스 부분 (조정 필요)

- 약관 종류 (`terms` 테이블 시드)
- 이메일 템플릿 (`email-verification`, `password-reset`)
- SMS 템플릿
- 패스워드 정책 (길이 / 유출 차단 사용 여부)
- enumeration 정책 (409 vs 200)
- 가입 후 ACTIVE 여부 (이메일 인증 강제 / 선택)
- Rate limit 수치
- providerType (소셜 사용 여부)

→ **각 회사 / 도메인 정책에 따라 조정**.

---

## 7. 관련

- [[signup|↑ hub]]
- [[prerequisites]] — 다음 (§2)
- [[design-decisions]] — 권장 도구 가이드
