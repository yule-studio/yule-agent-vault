---
title: "UserStatus — user 의 lifecycle 상태"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:32:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - enum
  - user-status
---

# UserStatus — user 의 lifecycle 상태

**[[enums|↑ enums hub]]**  ·  관련: [[../domain-model]]

> user 의 생애 주기 상태. **4개 (PENDING_VERIFICATION / ACTIVE / SUSPENDED / DELETED)**.
> 왜 4개인지, 왜 더 많지 않은지, 왜 더 적지 않은지의 design rationale.

---

## 1. 값 정의

```java
// src/main/java/com/example/shop/domain/user/UserStatus.java
public enum UserStatus {

    /** 가입 직후. 이메일 / 휴대폰 인증 대기. 로그인 불가. */
    PENDING_VERIFICATION,

    /** 정상 사용자. 로그인 OK / 모든 기능 사용 가능. */
    ACTIVE,

    /** 관리자가 정지. 로그인 불가. unsuspend 시 ACTIVE 복귀. */
    SUSPENDED,

    /** 탈퇴 (soft delete). 종착역 — 복구 X. */
    DELETED;

    // ---- query methods ----

    /** 로그인 시도 가능한가. */
    public boolean canLogin() { return this == ACTIVE; }

    /** 도메인 변경 (changePassword / changeName / ...) 가능한가. */
    public boolean canModify() { return this == ACTIVE || this == SUSPENDED; }

    /** 종착 상태. 추가 transition X. */
    public boolean isTerminal() { return this == DELETED; }

    /** 외부 (DTO) 응답에 표시할 라벨. i18n 필요하면 MessageSource 사용. */
    public String label() {
        return switch (this) {
            case PENDING_VERIFICATION -> "인증 대기";
            case ACTIVE -> "활성";
            case SUSPENDED -> "정지";
            case DELETED -> "탈퇴";
        };
    }
}
```

---

## 2. 왜 이 4개인가

### 2.1 PENDING_VERIFICATION

- **왜 필요**: 가입 직후 user 가 봇 / 가짜 이메일일 수 있음. 인증 거치기 전엔 신뢰 X.
- **대안**:
  - `UNVERIFIED` — 의미는 같지만 "verification 대기 중" 이 더 명확
  - `INACTIVE` — 너무 일반적 (정지된 user 와 구분 안 됨)
- **본 vault**: `PENDING_VERIFICATION` — "verify 를 기다리는 상태" 명확

### 2.2 ACTIVE

- **왜 필요**: 정상 사용자. 모든 기능 OK.
- **대안**:
  - `VERIFIED` — 인증만 됐지 정지 안 됨 표현 못 함
  - `ENABLED` — generic
- **본 vault**: `ACTIVE` — 가장 표준적

### 2.3 SUSPENDED

- **왜 필요**: 관리자가 일시 정지 (운영 정책 위반 / 부정 사용). 복구 가능.
- **대안**:
  - `BANNED` — 영구 정지 뉘앙스 (복구 X)
  - `RESTRICTED` — 일부 기능만 막는다는 뉘앙스 (의도 안 맞음)
- **본 vault**: `SUSPENDED` — "일시 정지, 풀 수 있음"

### 2.4 DELETED

- **왜 필요**: 사용자 탈퇴 — soft delete (FK 유지, 행 자체는 보존).
- **대안**:
  - `WITHDRAWN` — 한국 SaaS 에서 자주 사용 ("탈퇴")
  - `DEACTIVATED` — 일시 vs 영구 구분 모호
- **본 vault**: `DELETED` — 영문 표준 + GDPR / 한국 정책 의 "처리 완료" 뉘앙스

→ 관련: [[../database#4.2 Soft delete vs Hard delete]] — DELETE 시 email anonymize.

---

## 3. 왜 더 많지 않은가 — 추가 안 한 값들

### 후보: `LOCKED` (로그인 5회 실패 lock)

**왜 안 만드는가**: lock 은 **시간 기반** (15분 후 자동 해제). 상태로 두면 cron job 으로 해제 필요. **Redis TTL key** 로 표현이 훨씬 단순:

```
Key: login:fail:{ip}:{email}    (TTL 15분)
Value: 실패 횟수
```

→ [[../login-impl#7 Rate Limit — 로그인 5회 실패 lock]] 참고. status 폭증 방지.

### 후보: `EMAIL_VERIFIED` / `PHONE_VERIFIED` (인증 단계 별)

**왜 안 만드는가**: status 폭증 (`PENDING_EMAIL`, `PENDING_PHONE`, `EMAIL_VERIFIED`, `PHONE_VERIFIED`, `BOTH_VERIFIED`, ...). user 의 verify 진행 여부는 **별도 boolean 컬럼** (`email_verified_at`, `phone_verified_at`) 으로:

```sql
ALTER TABLE users
    ADD COLUMN email_verified_at TIMESTAMPTZ,
    ADD COLUMN phone_verified_at TIMESTAMPTZ;
```

`status = ACTIVE` 는 "기본 인증이 완료된 상태" 의미. 추가 인증은 timestamp 로.

### 후보: `INACTIVE` (장기 미접속)

**왜 안 만드는가**: GDPR 휴면 정책 적용 시 — `dormant_at` timestamp 컬럼 + scheduled job 으로 처리. status 까지 안 가도 됨.

### 후보: `MERGED` (account merging — 같은 사람의 2 계정 합침)

**왜 안 만드는가**: merging 은 별도 비즈니스 흐름. `DELETED` + `merged_into_user_id` 외래키로 추적.

---

## 4. 왜 더 적지 않은가 — 빠지면 안 되는 값들

### 옵션: PENDING_VERIFICATION 빼고 가입 직후 ACTIVE

```
가입 → ACTIVE → 이메일 인증 안 해도 사용 가능
```

**왜 안 하는가**:
- 가짜 이메일 가입 → 스팸 가입 폭증
- "이메일 인증 안 한 user" 메일링 / CRM 에서 혼란
- 사고 시 사용자 식별 불가 (이메일 무효)

→ **PENDING_VERIFICATION 필수**.

### 옵션: SUSPENDED 빼고 DELETE 만

**왜 안 하는가**: 부정 사용자 발견 시 즉시 DELETE = 정정 (오판단) 시 복구 X. **임시 정지 후 검토** 가 정석.

---

## 5. 상태 전이 매트릭스

| 현재 \ 다음 | PENDING_VERIFICATION | ACTIVE | SUSPENDED | DELETED |
| --- | --- | --- | --- | --- |
| **PENDING_VERIFICATION** | — | `verifyEmail()` ✅ | ❌ | `delete()` ✅ |
| **ACTIVE** | ❌ | — | `suspend()` ✅ | `delete()` ✅ |
| **SUSPENDED** | ❌ | `unsuspend()` ✅ | — | `delete()` ✅ |
| **DELETED** | ❌ | ❌ | ❌ | 멱등 (no-op) |

> 모든 transition 은 도메인 메서드 (`User.verifyEmail()` 등) 가 책임. enum 자체는 transition 모름 — 상태값만.

---

## 6. DB / API 표현

### DB 컬럼

```sql
status VARCHAR(30) NOT NULL,
-- DB constraint 로 값 강제 (옵션)
CONSTRAINT chk_user_status CHECK (
    status IN ('PENDING_VERIFICATION', 'ACTIVE', 'SUSPENDED', 'DELETED')
)
```

### JPA 매핑

```java
@Enumerated(EnumType.STRING)              // ⚠️ ORDINAL X (enum 순서 바뀌면 사고)
@Column(nullable = false, length = 30)
private UserStatus status;
```

### API 응답

```json
{
  "status": "PENDING_VERIFICATION",
  "statusLabel": "인증 대기"          // 옵션 (UI 직접 매핑이 더 일반적)
}
```

본 vault 기본: **값만** (label 은 클라가 매핑). admin / 사내 도구만 label 동시 제공.

---

## 7. 검색 / 필터링 활용

```java
// Admin — 인증 대기 + 활성 사용자만
@Query("select u from UserJpaEntity u where u.status in :statuses")
Page<UserJpaEntity> findByStatusIn(@Param("statuses") List<UserStatus> statuses, Pageable p);

// DELETED 자동 제외 (모든 read 의 default)
@Query("from UserJpaEntity u where u.status <> 'DELETED'")
List<UserJpaEntity> findAllActive();
```

partial index 활용:
```sql
CREATE INDEX ix_users_active ON users (created_at DESC) WHERE status <> 'DELETED';
```

→ DELETED 가 ~10% 까지 차도 인덱스 size 줄임.

---

## 8. 함정 모음

### 함정 1 — `@Enumerated(EnumType.ORDINAL)`
순서가 의미. enum 순서 바꾸면 모든 row 의 의미 깨짐. **반드시 STRING**.

### 함정 2 — DELETED user 의 email 로 재가입 막힘
`ux_users_email` UNIQUE 에 DELETED row 가 남아 있으면 재가입 불가. **delete 시 email anonymize** ([[../database#4.2]]).

### 함정 3 — 'INACTIVE' 같은 generic 이름
정지 vs 휴면 vs 미인증 구분 안 됨. **명확한 이름**.

### 함정 4 — 새 status 추가 시 모든 분기 깨짐
```java
switch (status) {
    case PENDING_VERIFICATION -> ...
    case ACTIVE -> ...
    // 새 LOCKED 추가하면 여기 누락
}
```
**switch expression + exhaustive check** (Java 21):
```java
return switch (status) {
    case PENDING_VERIFICATION, ACTIVE, SUSPENDED, DELETED -> ...
    // 새 값 추가 시 compile error → 모든 분기 강제 갱신
};
```

### 함정 5 — status 만 보고 로그인 가능 판단
`status == ACTIVE` 만으론 불충분 — locked 카운터 / 2FA 미통과 등 추가 검증. `canLogin()` 메서드 사용.

### 함정 6 — DB constraint 만 의존
application 단 (`UserStatus.valueOf("UNKNOWN")`) 시 IllegalArgumentException. **양쪽 검증**.

---

## 9. 관련

- [[enums|↑ enums hub]]
- [[../domain-model]] — User Aggregate 의 transition 메서드들
- [[verification-token-status]] · [[refresh-token-status]] — 토큰 lifecycle
- [[../database#4.2 Soft delete]] — DELETED 시 email anonymize
