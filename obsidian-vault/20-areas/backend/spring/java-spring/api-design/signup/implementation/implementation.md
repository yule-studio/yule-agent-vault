---
title: "auth §7 — 구현 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - implementation
  - hub
---

# auth §7 — 구현 (Hub)

**[[../signup|↑ signup hub]]**  ·  ← [[../security/security]]  ·  → [[../transactions]]

> auth 도메인의 6 핵심 흐름 + 1 모델 비교. 각 흐름의 "코드 + 왜 그렇게 작성했는가" 4구조.

---

## 1. 6 핵심 흐름

| 노트 | 무엇 | 의존성 |
| --- | --- | --- |
| [[signup-impl]] | 회원가입 — User 생성 + 약관 동의 + 이메일 인증 trigger | DB, outbox |
| [[login-impl]] | 로그인 — credential 검증 + JWT 발급 + refresh 생성 | DB |
| [[token-refresh-impl]] | refresh rotation + reuse detection | DB, Redis (옵션) |
| [[email-verification-impl]] | 이메일 인증 — token 발급 / 검증 | DB, SES |
| [[phone-verification-impl]] | SMS 6-digit 인증 | Redis, NCP SENS |
| [[password-reset-impl]] | 비밀번호 재설정 — 메일 link + 새 비밀번호 + 모든 RT revoke | DB, SES |

### 1.1 모델 비교

| 노트 | 무엇 |
| --- | --- |
| [[email-verification-model]] | URL token vs 6-digit code 비교 + 본 vault 선택 (URL token) |

---

## 2. 구현 순서 (의존성)

```
1. domain (VO + Aggregate)         [[../domain-model]]
   ↓
2. DB schema (Flyway)              [[../database]]
   ↓
3. signup-impl                     [[signup-impl]]
   ↓
4. email-verification-impl         [[email-verification-impl]]
   ↓
5. login-impl                      [[login-impl]]
   ↓
6. token-refresh-impl              [[token-refresh-impl]]
   ↓
7. phone-verification-impl         [[phone-verification-impl]]
   ↓
8. password-reset-impl             [[password-reset-impl]]
```

**왜 이 순서**
- signup 이 모든 후속 흐름의 시작점.
- email-verification = signup 의 직후 흐름 (user 가 ACTIVE 되어야 login 가능).
- login = email-verification 후.
- token-refresh = login 의 연장.
- phone-verification = signup 의 사전 단계 (옵션).
- password-reset = login 이후 별개 흐름.

자세히: [[../implementation-order]].

---

## 3. 공통 패턴 — 모든 흐름이 따르는

### 3.1 Controller → UseCase → Domain + Repository

```
[Controller]
  - HTTP I/O
  - DTO 검증 (Bean Validation)
  - UseCase 호출
  - 응답 변환

[UseCase] (@Service)
  - 비즈니스 흐름 조정
  - @Transactional 경계
  - 도메인 검증 (PasswordPolicy 등)
  - 도메인 객체 호출
  - Event publish

[Domain]
  - pure java
  - invariant 강제
  - 상태 전이
  - DomainEvent 발행

[Repository Port → Adapter]
  - 인터페이스는 도메인에
  - 구현은 infrastructure
```

자세히: [[../architecture]] · [[../domain-model]].

### 3.2 4계층 (Bean Validation → 도메인 VO → DB CHECK → DB UNIQUE)

| Layer | 책임 |
| --- | --- |
| Controller (Bean Validation) | 1차 — 형식 (`@Email`, `@Size`) |
| Domain VO | 2차 — 도메인 규칙 (`Email` record) |
| DB CHECK | 3차 — DB 안전망 (enum 외 값 차단) |
| DB UNIQUE | 4차 — race condition 차단 |

자세히: [[../security/sensitive-data-handling]] · [[../database/users-table#5 정합성]].

### 3.3 도메인 이벤트 + AFTER_COMMIT listener

```
[UseCase]
  @Transactional
  - 비즈니스 로직 + DB save
  - eventPublisher.publishEvent(new XxxEvent(...))
   ↓ COMMIT

[Listener] (@TransactionalEventListener(phase = AFTER_COMMIT))
  - outbox INSERT (별도 트랜잭션)
  - 메트릭
  - 로그
```

**왜 이 패턴**
- 트랜잭션 안에서 외부 호출 X — 락 / 연결 점유 회피.
- 비즈니스 로직 / 부수 효과 분리.
- 새 후속 처리 = 새 listener (UseCase 수정 X).

자세히: [[../transactions]] · [[../database/email-outbox-table]].

---

## 4. 공통 의존성

```kotlin
// build.gradle.kts (auth 도메인 필요한 부분)
implementation("org.springframework.boot:spring-boot-starter-web")
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
implementation("org.springframework.boot:spring-boot-starter-validation")
implementation("org.springframework.boot:spring-boot-starter-security")

implementation("com.password4j:password4j:1.8.2")              // Argon2id
implementation("io.jsonwebtoken:jjwt-api:0.12.6")              // JWT
implementation("com.github.f4b6a3:ulid-creator:5.2.3")         // ULID
implementation("software.amazon.awssdk:sesv2:2.25.50")         // SES
```

자세히: [[../design-decisions/dependencies]].

---

## 5. 응답 / 에러 표준

모든 응답은 `CommonResponse<T>` envelope:

```json
{
  "code": "OK_001",
  "message": "회원가입 성공",
  "data": { "userId": "01HQ...", "email": "...", ... }
}
```

자세히: [[../../common/response-envelope]].

---

## 6. 관련

- [[../signup|↑ signup hub]]
- [[../security/security]] — 이전 (§6)
- [[../transactions]] — 다음 (§8)
- [[../domain-model/domain-model]] · [[../database/database]]
- [[../design-decisions/design-decisions]]
