---
title: "auth §6 — 도메인 모델 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - domain-model
  - hub
---

# auth §6 — 도메인 모델 (Hub)

**[[../signup|↑ signup hub]]**  ·  ← [[../database/database|database]]  ·  → [[../architecture|architecture]]

> auth 도메인의 객체·규칙 모음.
> Aggregate / Value Object / Domain Event / Repository port — 각자 자기 노트로 분리.

---

## 1. 이 폴더의 노트

| 노트 | 무엇 |
| --- | --- |
| [[user-aggregate]] | User Aggregate Root — 메서드 / transition / 책임 |
| [[value-objects]] | Value Object 패턴 — 왜 record / compact constructor / 4종 VO 소개 |
| [[email-vo]] | `Email` — RFC 5322 / 정규화 위치 |
| [[password-hash-vo]] | `PasswordHash` — argon2 PHC 형식 검증 |
| [[user-id-vo]] | `UserId` — ULID 선택 이유 |
| [[phone-number-vo]] | `PhoneNumber` — 한국 휴대폰 / 정규화 / E.164 |
| [[domain-events]] | 5 events + listener + AFTER_COMMIT |
| [[repository-ports]] | UserRepository / RefreshTokenRepository / VerificationTokenRepository ports |
| [[domain-rules]] | 9 불변식 + 각 규칙의 책임 위치 |
| [[aggregate-boundaries]] | Aggregate 경계 결정 — 왜 User 가 약관 동의를 안 가지나 |

→ enums (UserStatus / RefreshTokenStatus / ...) 는 [[../enums/enums|enums/]] 별도 폴더.

---

## 2. 도메인 그림 (high-level)

```
┌─────────────────────────────────────────┐
│  User (Aggregate Root)                  │
│  - id: UserId          ←──── ULID       │
│  - email: Email        ←──── Value Obj  │
│  - phone: Phone (옵션)                  │
│  - passwordHash: PasswordHash           │
│  - name: String                         │
│  - status: UserStatus  ←──── enum       │
│  - role: Role          ←──── enum       │
│  - providerType: SocialProviderType     │
│  - createdAt: Instant                   │
│                                          │
│  + register(...) → static               │
│  + verifyEmail()                        │
│  + verifyPhone()                        │
│  + changePassword(...)                  │
│  + suspend() / unsuspend() / delete()   │
│                                          │
│  raises DomainEvent ──→  ┐              │
└──────────────────────────┼──────────────┘
                            │
                            ▼
                ┌──────────────────────┐
                │ UserRegistered       │
                │ UserEmailVerified    │
                │ UserPhoneVerified    │
                │ UserPasswordChanged  │
                │ UserSuspended        │
                └──────────────────────┘

┌─────────────────────────────────────────┐
│  RefreshToken (Aggregate)               │
│  - id: RefreshTokenId (jti)             │
│  - userId: UserId (외부 Aggregate 참조)   │
│  - tokenHash: String                    │
│  - status: RefreshTokenStatus           │
│  - expiresAt: Instant                   │
│  - rotatedToId: RefreshTokenId?          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  VerificationToken (3 종)                │
│  - EmailVerificationToken               │
│  - PhoneVerificationCode (6-digit + lock)│
│  - PasswordResetToken                   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Repository Ports (domain interface)    │
│  - UserRepository                       │
│  - RefreshTokenRepository               │
│  - EmailVerificationTokenRepository     │
│  - PhoneVerificationCodeRepository      │
│  - PasswordResetTokenRepository         │
└─────────────────────────────────────────┘
```

---

## 3. 상태 머신 (high-level)

```
[User]
   PENDING_VERIFICATION ──verifyEmail──→ ACTIVE ──suspend──→ SUSPENDED ──unsuspend──→ ACTIVE
                                            │                    │
                                            └─────── delete ─────┴──→ DELETED (종착)

[RefreshToken]
   ACTIVE ──rotate──→ ROTATED          ←─ reuse 감지 시
   ACTIVE ──revoke──→ REVOKED          ←─ logout / 패스워드 변경
   ACTIVE ──expire──→ EXPIRED          ←─ TTL 지남

[VerificationToken]
   ACTIVE ──consume──→ USED            ←─ 정상 사용
   ACTIVE ──revoke ──→ REVOKED         ←─ 재발송
   ACTIVE ──expire ──→ EXPIRED         ←─ TTL
```

각 상태 / 전이 상세: [[../enums/user-status]] · [[../enums/refresh-token-status]] · [[../enums/verification-token-status]].

---

## 4. 의존 관계

```
Aggregate Root (User, RefreshToken, VerificationToken)
       │ 사용
       ▼
Value Objects (Email, PasswordHash, UserId, PhoneNumber)
       │ raises
       ▼
Domain Events (UserRegistered, ...)
       │ 구독 (application / infra)
       ▼
Listener (이메일 outbox / Kafka / 푸시 알림)

Repository (port) ← Adapter (JPA / MyBatis) — 도메인이 모르는 곳
```

**도메인 layer 는**:
- `jakarta.validation`, `java.time`, `java.util` 만 import
- Spring / JPA / MyBatis 의존 0
- HTTP / DB / 외부 API 모름
- 단위 테스트가 ms 단위

---

## 5. 코드 컨벤션 (이 폴더 전체)

- **record** 우선 (Value Object / Event)
- **final class + 메서드** (Aggregate)
- **compact constructor** 에서 검증
- **public 생성자 X** → static factory (`User.register(...)`)
- **setter X** → 의미 있는 메서드 (`verifyEmail()` 같은)
- **`pullDomainEvents()`** — 이벤트는 한 번만 가져감 (idempotent X — caller 책임)
- **`reconstitute()`** — Adapter 의 JPA → 도메인 매핑 전용 static

자세한 예 — [[user-aggregate]].

---

## 6. 관련

- [[../signup|↑ signup hub]]
- [[../enums/enums|↗ enums/]] — 모든 enum
- [[../architecture]] — 계층 / Port-Adapter 흐름
- [[../database/database|↗ database/]] — DB schema (참조)
