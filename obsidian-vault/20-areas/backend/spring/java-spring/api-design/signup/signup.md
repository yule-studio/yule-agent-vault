---
title: "회원가입 (Signup) — Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - signup
  - hub
---

# 회원가입 (Signup) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-14 | engineering-agent/tech-lead | v2 표준 목차 적용 / 폴더 split (12 detail files) |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | v1 단일 파일 (deprecated) |

**[[../api-design|↑ api-design hub]]**

> 📐 **ORM**: 본 폴더의 §6 implementation 은 JPA Adapter sketch. 3 모드 정책: [[../api-design#0.5 ORM 정책]].
> ORM 변형 본거지 — JPA: [[../../database/jpa#11.5.1]] · MyBatis: [[../../database/mybatis#10.5.1]] · 공존: [[../../database/jpa-mybatis-coexist#9.5.1]]
> 공통 패턴: [[../../common/response-envelope]] · [[../../common/security-config]]

---

## 1. 한 줄 요약

`POST /api/v1/auth/signup` — 이메일 + 패스워드 + 이름 + 약관 동의 → 신규 user 생성 (`PENDING_VERIFICATION` 상태). 부수효과: 이메일 인증 메일 outbox 적재 ([[../email-verification]]).

---

## 2. 전체 흐름 한눈에

```
[Client]
  POST /api/v1/auth/signup
   ↓
[Controller]  ────  Bean Validation
   ↓
[Application — SignupUseCase]
   │ @Transactional 시작
   ├─ PasswordPolicy.validate
   ├─ Email 정규화 (lowercase trim)
   ├─ users.existsByEmail (1차 검증)
   ├─ User.register(...)          ← 도메인 (PENDING_VERIFICATION + UserRegistered event)
   ├─ users.save                  ← DB UNIQUE (lower(email)) 가 진실의 원천
   └─ events.publishAll           ← in-memory 큐잉
   │ @Transactional 커밋
   ↓
[Listener — AFTER_COMMIT]
   └─ EmailOutboxRepository.enqueue   ← 별도 워커가 SMTP 발송
   ↓
HTTP 200 + CommonResponse(OK_001, "회원가입 성공", { userId, email, status, createdAt })
```

---

## 3. 13 섹션 목차 — 어디부터 읽나

| 섹션 | 노트 | 한 줄 |
| --- | --- | --- |
| 0. 전제 / 범위 | [[prerequisites]] | 언제 이 레시피 적용 / 과한 적용 기준 |
| 1. 무엇을 만드는가 + 완료 조건 | [[requirements]] | API spec + Acceptance Criteria |
| 2. 도메인 모델 + 규칙 + 상태 전이 | [[domain-model]] | User Aggregate / Email VO / 상태머신 |
| 3. 아키텍처 / Port·Adapter | [[architecture]] | 계층 책임 + 의존성 흐름 |
| 4. DB 스키마 + 조회 패턴 + 정합성 | [[database]] | users 테이블 + lower(email) UNIQUE |
| 5. 보안 / 인증·인가 / 암호화 | [[security]] | anonymous endpoint / argon2id |
| 6. 구현 — Java | [[implementation]] | 패키지 구조 + DTO + UseCase + Controller + JPA |
| 7. 트랜잭션 + 동시성 + 멱등성 | [[transactions]] | @Transactional + DataIntegrityViolation + Idempotency-Key |
| 8. 테스트 | [[testing]] | 시나리오 표 + 단위 + 통합 (Testcontainers) |
| 9. 운영 체크리스트 | [[operations]] | 배포 / 로그 / 메트릭 / 알림 |
| 10. 구현 순서 | [[implementation-order]] | 단계별 to-do (요구사항 → DB → port → ...) |
| 11. 흔한 함정 | [[pitfalls]] | 안티패턴 모음 |

---

## 4. 한 페이지 cheat sheet

### 4.1 도메인 객체

```java
User                          (Aggregate Root)
├── id: UserId (ULID)
├── email: Email              (Value Object — RFC 5322 검증)
├── passwordHash: PasswordHash (Value Object — argon2 hash 만)
├── name: String (1~100)
├── status: UserStatus         (PENDING_VERIFICATION → ACTIVE → SUSPENDED → DELETED)
└── createdAt: Instant
   ↑
   raises: UserRegistered, UserEmailVerified, UserPasswordChanged
```

### 4.2 상태 전이

```
[가입]
  ↓
PENDING_VERIFICATION ──── verifyEmail ────→ ACTIVE
                                                ↓
                                             changePassword(...)
                                             changeName(...)
                                                ↓
                                            SUSPENDED  (관리자)
                                                ↓
                                            DELETED (soft, status only)

* 어느 상태에서도 ACTIVE 로 직접 못 감 — 인증을 거쳐야.
* DELETED → 복구 X (정책)
```

### 4.3 의존성 흐름

```
SignupController
   ↓ (call)
SignupUseCase                  ← @Transactional 경계
   ↓
UserRepository (port, domain/)
   ↑ implements
JpaUserRepositoryAdapter (infrastructure/persistence/jpa/)
   ↓ uses
UserJpaRepository (Spring Data)
```

---

## 5. 관련

- [[../login-jwt]] — 가입 후 로그인
- [[../password-reset]] — 잊은 패스워드
- [[../email-verification]] — `PENDING_VERIFICATION → ACTIVE` 전이
- [[../oauth2-social-login]] — 소셜 가입
- [[../two-factor-auth]] — 2FA
- [[../rbac-permissions]] — 가입 후 role
- [[../../../../database/postgresql/security|↗ PG 보안]]
- [[../../../../security/security|↗ security hub]]
- [[../api-design|↑ api-design hub]]
