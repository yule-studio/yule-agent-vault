---
title: "signup §1 — 무엇을 만드는가 + 완료 조건"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:34:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - requirements
  - acceptance-criteria
---

# signup §1 — 무엇을 만드는가 + 완료 조건

**[[signup|↑ signup hub]]**  ·  ← [[prerequisites]]  ·  → [[domain-model]]

---

## 1. API spec

### 1.1 Request

```http
POST /api/v1/auth/signup
Content-Type: application/json

{
  "email": "alice@example.com",
  "password": "Tr0ub4dor&3-with-len",
  "name": "Alice",
  "termsAgreed": true,
  "marketingAgreed": false
}
```

### 1.2 Response — 성공

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "code": "OK_001",
  "status": "OK",
  "message": "회원가입 성공",
  "result": {
    "userId": "01HZ9X...",
    "email": "alice@example.com",
    "status": "PENDING_VERIFICATION",
    "createdAt": "2026-05-14T09:15:23+09:00"
  }
}
```

### 1.3 Response — 실패 (예시 4종)

```http
409 Conflict — 중복 이메일
{ "code": "BADREQ_004", "status": "DUPLICATE_DATA",
  "message": "이미 가입된 이메일입니다." }

422 Unprocessable Entity — 약한 패스워드
{ "code": "BADREQ_002", "status": "INVALID_INPUT_FORMAT",
  "message": "password length must be 8-128" }

400 Bad Request — Bean Validation 실패
{ "code": "BADREQ_002", "status": "INVALID_INPUT_FORMAT",
  "message": "validation failed",
  "result": { "email": "must be email", "termsAgreed": "must agree to terms" } }

500 Internal Server Error — 그 외
{ "code": "INTERNAL_ERR_001", "status": "INTERNAL_SERVER_ERROR",
  "message": "서버 내부 오류가 발생했습니다." }
```

### 1.4 비기능 (NFR)

| 항목 | 기준 |
| --- | --- |
| latency p95 | < 500ms (argon2id 가 100~300ms 차지) |
| 이메일 unique | DB constraint + 비즈니스 검증 이중 |
| 평문 패스워드 | 어디에도 저장 / 로그 X (입력 즉시 해시) |
| 이메일 발송 | 트랜잭션 밖 (outbox + AFTER_COMMIT) |
| Idempotency | (옵션) `Idempotency-Key` 헤더 — 동일 키 24시간 같은 응답 |
| 동시성 | 같은 이메일 동시 가입 시 1 명만 성공 |
| 가용성 | 99.5% (auth 는 critical path) |

---

## 2. 완료 조건 (Acceptance Criteria)

### 2.1 정상 성공

**Given**: 가입되지 않은 이메일 `alice@x.com` + 강한 패스워드 + 약관 동의

**When**: `POST /api/v1/auth/signup` 호출

**Then**:
- 응답 200 + `code=OK_001`
- DB `users` 테이블에 row 1개 추가
  - `status = 'PENDING_VERIFICATION'`
  - `password_hash` 가 `$argon2id$...` 로 시작 (평문 X)
  - `email` 은 소문자 정규화 (`alice@x.com` 그대로)
- `user_terms_consent_history` 에 약관별 동의 row 추가 (필수 + 명시적 선택)
- DB 커밋 후 `email_outbox` 테이블에 `template=verification` row 적재
- 별도 워커가 이메일 발송 (응답 시간엔 영향 X)

### 2.2 중복 이메일

**Given**: 이미 `alice@x.com` 으로 가입된 user 존재

**When**: 같은 이메일로 가입 재시도

**Then**:
- 응답 409 + `code=BADREQ_004`
- 새 row 생성 X
- `email_outbox` 적재 X
- (정책 옵션) **enumeration 차단 모드**: 응답 200 + "이메일을 확인해 주세요" 동일 메시지

### 2.3 약한 패스워드 / 짧은 패스워드 / 유출 패스워드

**Given**: `password = "12345"` 또는 알려진 유출 패스워드

**When**: 가입 요청

**Then**:
- 응답 422 + `code=BADREQ_002`
- `result.message` 가 어떤 검증이 실패했는지 설명 (길이 / 공백 / 유출 여부)
- DB row 생성 X

### 2.4 약관 미동의

**Given**: `termsAgreed = false`

**When**: 가입 요청

**Then**:
- 응답 400 + Bean Validation 에러
- `result.termsAgreed = "must agree to terms"`

### 2.5 이메일 대소문자

**Given**: `Alice@X.com` 으로 가입 후, `alice@x.com` 으로 재가입 시도

**When**: 두 번째 가입 요청

**Then**:
- 응답 409 (`DUPLICATE_DATA`)
- DB 의 `users.email` 은 첫 가입 시 정규화된 `alice@x.com` 그대로

### 2.6 동시성 — 같은 이메일 2 요청

**Given**: 두 개의 동시 HTTP 요청이 같은 이메일로 옴 (race)

**When**: 둘 다 1차 검증 (existsByEmail) 통과 후 둘 다 INSERT 시도

**Then**:
- 1 개만 성공 (200)
- 나머지 1 개는 `DataIntegrityViolationException` → 409 변환
- DB row 1개만 (정확히)

### 2.7 동일 요청 반복 (멱등 — `Idempotency-Key` 사용 시)

**Given**: 같은 `Idempotency-Key` 헤더로 2번 가입 요청 (네트워크 retry)

**When**: 첫 응답 후 즉시 재시도

**Then**:
- 둘 다 응답 200 + 동일한 `userId`
- 새 row 생성 X (멱등)
- 이메일 outbox row 도 1개만

### 2.8 Bean Validation 실패

**Given**: `email = "invalid"` 또는 `password = "short"` 또는 `name = ""`

**When**: 가입 요청

**Then**:
- 응답 422 + `code=BADREQ_002`
- `result` 에 필드 별 에러 (`{ "email": "must be email", "password": "size must be 8-128" }`)

### 2.9 응답에 평문 패스워드 / 해시 노출 X

**Given**: 가입 성공

**Then**:
- 응답 JSON 에 `password` / `passwordHash` 필드 절대 없음
- 액세스 로그 / Hibernate SQL 로그에도 평문 password 노출 X

### 2.10 DB 가 일시 다운

**Given**: 가입 요청 중 DB connection pool 고갈

**When**: 요청 처리

**Then**:
- 응답 500 + `code=INTERNAL_ERR_002` (`DATABASE_ERROR`)
- `email_outbox` 적재 X
- 클라가 재시도 가능

---

## 3. Out of scope (이 API 가 안 하는 것)

- 자동 로그인 — 응답에 JWT 포함 X (별도 `/auth/login` 호출)
- 이메일 인증 메일 발송 자체 — outbox 적재만, 발송은 워커 / [[../email-verification]]
- 가입 직후 ACTIVE — `PENDING_VERIFICATION` 으로만
- 프로필 사진 / 추가 정보 — 별도 `/me/profile` endpoint
- KYC / 신원 인증 — 별도 도메인

---

## 4. 관련

- [[signup|↑ signup hub]]
- [[prerequisites]] — 이전 (§0)
- [[domain-model]] — 다음 (§2)
