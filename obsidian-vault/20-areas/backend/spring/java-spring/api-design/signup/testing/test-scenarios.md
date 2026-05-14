---
title: "Test Scenarios — 시나리오 표 + Given-When-Then"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - testing
  - scenarios
---

# Test Scenarios — 시나리오 표 + Given-When-Then

**[[testing|↑ testing hub]]**

> 어떤 시나리오를 어떤 우선순위로 테스트할지. [[../requirements#2]] 의 완료 조건 매핑.

---

## 1. 우선순위

| 마크 | 의미 | 시점 |
| --- | --- | --- |
| 🔴 必 | 출시 전 통과 필수 | PR 머지 전 |
| 🟡 권장 | 권장 (점진 추가) | sprint 안 |
| 🟢 옵션 | 시간 되면 | 안정 후 |

---

## 2. Signup 시나리오 (13)

| # | 시나리오 | 입력 | 기대 | 종류 | 우선 |
| --- | --- | --- | --- | --- | --- |
| 1 | 정상 가입 | 유효 email + 강한 password + 약관 동의 | 200, status=PENDING_VERIFICATION, DB row 1, outbox row 1 | 통합 | 🔴 |
| 2 | 약한 password (8자 미만) | password="short" | 422, BADREQ_002, DB 변동 X | 단위+통합 | 🔴 |
| 3 | 잘못된 email 형식 | email="not-an-email" | 422, BADREQ_002 (Bean Validation) | 단위 (Controller) | 🔴 |
| 4 | 약관 미동의 | termsAgreed=false | 400, AssertTrue 위반 | 단위 (Controller) | 🔴 |
| 5 | 중복 이메일 | 이미 가입된 email | 409, BADREQ_004, 새 row X | 통합 | 🔴 |
| 6 | 대소문자 다른 이메일 | "Alice@X.com" 후 "alice@x.com" | 409 (case-insensitive UNIQUE) | 통합 | 🔴 |
| 7 | 도메인 이벤트 발행 | 정상 가입 | UserRegistered 1번 publish | 단위 (UseCase) | 🟡 |
| 8 | 동시성 race | 같은 email 2개 동시 | 정확 1개 성공, 1개 409 | 통합 (멀티스레드) | 🟡 |
| 9 | 평문 password 노출 X | 정상 가입 | 응답 JSON 에 password 없음 | 통합 | 🔴 |
| 10 | toString 마스킹 | request 로깅 | `password=***` | 단위 | 🟢 |
| 11 | 약관 history 저장 | 정상 가입 | user_terms_consent_history row 추가 | 통합 | 🟡 |
| 12 | outbox AFTER_COMMIT | 정상 가입 | email_outbox row 추가, rollback 시 X | 통합 | 🟡 |
| 13 | Idempotency-Key 재사용 | 같은 key 2번 | 둘 다 200, userId 동일 | 통합 | 🟢 |

---

## 3. Login 시나리오

| # | 시나리오 | 기대 | 우선 |
| --- | --- | --- | --- |
| L1 | 정상 로그인 | 200, access + refresh 발급 | 🔴 |
| L2 | 잘못된 비밀번호 | 401, INVALID_CREDENTIALS | 🔴 |
| L3 | 존재하지 않는 email | 401 (timing 같음) | 🔴 |
| L4 | DELETED user | 401 | 🔴 |
| L5 | SUSPENDED user | 401, 메시지 분기 | 🔴 |
| L6 | PENDING_VERIFICATION user | 403, "이메일 인증 후 로그인" | 🔴 |
| L7 | 5회 실패 후 lock | 423 LOCKED, 15분 lock | 🔴 |
| L8 | Account lock 후 정상 로그인 | 30분 후 = 200, 15분 안 = 423 | 🟡 |
| L9 | 동시 로그인 (다중 디바이스) | 모두 성공, 각 다른 refresh | 🔴 |
| L10 | Timing attack 방어 | user 없음 vs 비번 틀림 응답 시간 차이 < 50ms | 🟡 |
| L11 | rate limit (IP) | 5회/min 초과 시 429 | 🔴 |

---

## 4. Token Refresh 시나리오

| # | 시나리오 | 기대 | 우선 |
| --- | --- | --- | --- |
| R1 | 정상 refresh | 200, 새 access + 새 refresh, 옛 refresh = ROTATED | 🔴 |
| R2 | 만료된 refresh | 401, EXPIRED | 🔴 |
| R3 | REVOKED refresh | 401, REVOKED | 🔴 |
| R4 | ROTATED refresh (reuse detection) | chain 전체 revoke + 401 | 🔴 |
| R5 | 잘못된 refresh format | 401, INVALID | 🔴 |
| R6 | refresh 의 user 가 DELETED | 401 | 🔴 |
| R7 | 동시 refresh (같은 refresh) | 하나만 성공, 나머지 401 | 🟡 |

---

## 5. Email Verification 시나리오

| # | 시나리오 | 기대 | 우선 |
| --- | --- | --- | --- |
| E1 | 정상 클릭 | 200, user.status = ACTIVE | 🔴 |
| E2 | 만료된 토큰 (24h 후) | 401, EXPIRED | 🔴 |
| E3 | 이미 사용된 토큰 (USED) | 401, USED | 🔴 |
| E4 | revoked (재발송 후 옛 link) | 401, REVOKED | 🔴 |
| E5 | 잘못된 토큰 | 401, INVALID | 🔴 |
| E6 | 재발송 — 옛 토큰 revoke | 옛=REVOKED, 새=ACTIVE | 🔴 |
| E7 | rate limit (1h 3회) | 4번째 시도 429 | 🔴 |
| E8 | ACTIVE user 재인증 시도 | 200, no-op | 🟡 |

---

## 6. Phone Verification 시나리오

| # | 시나리오 | 기대 | 우선 |
| --- | --- | --- | --- |
| P1 | 정상 코드 입력 | 200, phoneAuthToken 발급 | 🔴 |
| P2 | 잘못된 코드 | 401, attempts++ | 🔴 |
| P3 | 5회 실패 후 lock | 423, 새 코드 발급 강제 | 🔴 |
| P4 | 만료된 코드 (3분) | 401, EXPIRED | 🔴 |
| P5 | 정규화 — `010-1234-5678` vs `01012345678` | 같은 phone | 🔴 |
| P6 | 잘못된 phone 형식 | 422 | 🔴 |
| P7 | 60s cooldown | 60s 안 재요청 시 429 | 🟡 |
| P8 | rate limit (1h 5회) | 6번째 429 | 🔴 |

---

## 7. Password Reset 시나리오

| # | 시나리오 | 기대 | 우선 |
| --- | --- | --- | --- |
| PR1 | reset 요청 — 가입 user | 200, 메일 발송 | 🔴 |
| PR2 | reset 요청 — 미가입 (일반 SaaS) | 404 | 🔴 |
| PR2' | reset 요청 — 미가입 (금융 obscured) | 200 동일 응답 | 🟡 |
| PR3 | reset 토큰 클릭 → password 변경 | 200, password_hash 변경, RT revoke | 🔴 |
| PR4 | 만료된 reset 토큰 (30분) | 401 | 🔴 |
| PR5 | 이미 사용된 토큰 | 401 | 🔴 |
| PR6 | rate limit (1h 3회) | 4번째 429 | 🔴 |
| PR7 | password 변경 후 모든 RT revoke | 옛 RT 로 refresh 시도 = 401 | 🔴 |
| PR8 | pwned check | pwned password 시 422 | 🔴 |

---

## 8. Given-When-Then 작성법

```java
@Test
void signup_with_duplicate_email_returns_409() {
    // Given — 이미 가입된 user
    userRepo.save(newUserEntity("alice@example.com"));

    // When — 같은 email 로 가입 시도
    var response = mockMvc.perform(...);

    // Then — 409 + DB 새 row X
    response.andExpect(status().isConflict())
            .andExpect(jsonPath("$.code").value("BADREQ_004"));
    assertThat(userRepo.findAll()).hasSize(1);
}
```

### 8.1 왜 이 구조

- 가독성 — 명확 분리.
- 같은 패턴 반복 = 신규 합류자 학습 곡선 ↓.

### 8.2 Given 의 setup

- minimal — 시나리오 필요 데이터만.
- TestUserBuilder 활용.

자세히: [[test-data-fixtures]].

---

## 9. 시나리오 매트릭스 — 무엇이 covered

```
시나리오            | 단위 | 통합 | E2E |
--------------------|------|------|-----|
정상 가입           |  ✓   |  ✓   |  ✓  |
중복 이메일         |  ✓   |  ✓   |     |
약한 비번           |  ✓   |  ✓   |     |
race condition      |      |  ✓   |     |
toString 마스킹     |  ✓   |      |     |
outbox AFTER_COMMIT |      |  ✓   |     |
이메일 발송 (SES)   |      |      |  ✓  |
실제 SMS 발송       |      |      |  ✓  |
```

**왜 layer 별 분리**
- 단위 = pure logic.
- 통합 = JPA + DB + 트랜잭션.
- E2E = 외부 API (SES / SMS) 진짜 호출 (sandbox).

---

## 10. CI/CD 통합

```yaml
# .github/workflows/ci.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: ./mvnw test -Dgroups=unit

  integration:
    runs-on: ubuntu-latest
    steps:
      - run: ./mvnw test -Dgroups=integration
    services:
      docker: { image: docker:dind }

  e2e:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'        # main 만
    steps:
      - run: ./mvnw test -Dgroups=e2e
```

**왜 분리**
- 빠른 feedback — unit 먼저.
- 비용 절감 — E2E 는 main 만.

---

## 11. 함정 모음

### 함정 1 — 시나리오 표 없이 코드부터
무엇 테스트할지 모름 → coverage 빈 곳.
→ 표 먼저, 코드는 후.

### 함정 2 — 우선순위 없이 모두 동등
시간 부족 시 무엇 빼야 할지 모름.
→ 🔴/🟡/🟢 명시.

### 함정 3 — Happy path 만
edge case / 실패 시나리오 누락.
→ 각 endpoint 마다 정상 + 실패 다.

### 함정 4 — 시나리오 이름이 `test1`
무엇 검증인지 불명.
→ `signup_with_X_returns_Y` 형식.

### 함정 5 — Given-When-Then 구조 없음
가독성 ↓.
→ 주석 또는 빈 줄로 명시.

### 함정 6 — 시나리오 마다 다른 fixture
일관성 ↓.
→ TestUserBuilder + helper.

### 함정 7 — 응답 검증만 (DB 검증 X)
side-effect 미검증.
→ 응답 + DB 둘 다.

### 함정 8 — 실패 시나리오의 side-effect 검증 X
"실패 응답 받았으니 OK" — 실제 DB 변동 가능.
→ DB 검증 (`hasSize(0)` 같은).

### 함정 9 — E2E 모든 시나리오
시간 폭증.
→ critical path 만.

### 함정 10 — 시나리오 와 코드 sync 안 됨
표 업데이트 안 함 → 부정확.
→ 코드 옆에 표 명시 (또는 codegen).

### 함정 11 — Coverage 100% 만 추구
"통과는 했지만 검증은 X" — 가짜 coverage.
→ mutation testing.

### 함정 12 — flaky 시나리오 ignore
silent 실패 → 실제 버그 놓침.
→ flaky 발견 시 분석 + 수정 (suppress 금지).

---

## 12. 관련

- [[testing|↑ testing hub]]
- [[unit-tests]] · [[integration-tests]] · [[test-data-fixtures]]
- [[../requirements#2 완료 조건]] — 시나리오 매핑
- [[../signup-impl]] · [[../login-impl]] · 기타 구현
