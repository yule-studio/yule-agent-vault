---
title: "users 테이블 — schema + 인덱스 + soft delete"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:02:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - users-table
---

# users 테이블 — schema + 인덱스 + soft delete

**[[database|↑ database hub]]**

> auth 도메인의 핵심 table. 모든 다른 table 이 `user_id` 로 참조.
> 이 한 테이블의 설계 실수는 **전체 서비스의 보안 / 정합성 / 마이그레이션 비용** 으로 직결되므로 컬럼 단위로 "왜" 를 정리한다.

---

## 1. Schema

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id                  CHAR(26) PRIMARY KEY,                  -- ULID
    email               VARCHAR(254) NOT NULL,                  -- RFC 5321 max
    phone               VARCHAR(20),                            -- 정규화 후 (01012345678)
    phone_hash          CHAR(64),                                -- SHA-256 (검색용, 옵션)
    password_hash       VARCHAR(255),                            -- argon2 PHC, 소셜은 null
    name                VARCHAR(100) NOT NULL,
    status              VARCHAR(30)  NOT NULL,
    role                VARCHAR(20)  NOT NULL DEFAULT 'USER',
    provider_type       VARCHAR(20)  NOT NULL DEFAULT 'LOCAL',
    external_id         VARCHAR(100),                            -- 소셜 provider 의 user ID
    apple_sub           VARCHAR(100),                            -- Apple 만
    email_verified_at   TIMESTAMPTZ,
    phone_verified_at   TIMESTAMPTZ,
    version             BIGINT NOT NULL DEFAULT 0,               -- @Version 낙관 락
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_users_status
        CHECK (status IN ('PENDING_VERIFICATION', 'ACTIVE', 'SUSPENDED', 'DELETED')),
    CONSTRAINT chk_users_role
        CHECK (role IN ('USER', 'SELLER', 'ADMIN', 'MASTER')),
    CONSTRAINT chk_users_provider
        CHECK (provider_type IN ('LOCAL', 'APPLE', 'GOOGLE', 'KAKAO', 'NAVER')),
    CONSTRAINT chk_users_local_has_password
        CHECK (provider_type <> 'LOCAL' OR password_hash IS NOT NULL)
);
```

---

## 2. 각 컬럼의 "왜" — 4구조 (왜 필요 / 안 하면 문제 / 대안 / 트레이드오프)

### 2.1 `id CHAR(26) PRIMARY KEY` — ULID

**왜 필요한가**
- 모든 다른 table (refresh_tokens, verification_tokens, orders, ...) 의 FK 대상.
- 사용자 식별의 **불변 식별자**. email / phone 은 변경 가능하지만 id 는 절대 안 바뀜.
- API 응답 / URL 에 그대로 노출 가능해야 함 (`GET /v1/users/{id}/me`).

**안 하면 무슨 문제**
- `email` 을 PK 로 쓰면 → email 변경 시 모든 FK row 가 같이 변경 (cascade update 폭탄).
- `BIGSERIAL` (auto-increment) 면 → `GET /users/12345` 다음 ID 추측 가능 (사용자 수 / 가입 순서 노출).
- UUID v4 면 → INSERT 시 B-tree page split 빈발 (대규모 시 인덱스 fragmentation, INSERT throughput ~30% ↓ 사례 흔함).

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| `BIGSERIAL` | 순차 노출 + 분산 생성 불가 + MSA 분리 시 ID 충돌 |
| `UUID v4` (random) | B-tree fragmentation, 36 char (ULID 26 보다 김) |
| `UUID v7` | 좋은 대안 (시간순). ULID 와 거의 동등 — 본 vault 는 도구 성숙도/26 char 로 ULID 선택 |
| `Snowflake (64bit)` | 워커 ID 관리 인프라 부담, ZooKeeper 의존 사례 |

**트레이드오프**
- ULID 의 timestamp 부분은 노출 → 생성 시각 추정 가능. 단 `created_at` 컬럼이 이미 있으므로 추가 정보 누출 0.
- 16 byte 저장 (Long 의 2배). users 1000만 행도 추가 ~80MB 수준 — 무시 가능.

자세히: [[id-strategy]].

---

### 2.2 `email VARCHAR(254) NOT NULL`

**왜 필요한가**
- 로그인 / 비밀번호 재설정 / 알림 발송의 primary contact.
- 254 = **RFC 5321 의 path 길이 한계** (local 64 + `@` + domain 255 - 1 leading char) — 실제 운용 가능한 최대치.

**안 하면 무슨 문제**
- `VARCHAR(255)` 로 잡으면 → 256 byte 짜리 이메일 INSERT 시 RFC 위반인데 DB 는 통과. 다른 시스템 (SES, SMTP relay) 에서 reject → 발송 실패가 회원가입 성공 후 발생.
- `VARCHAR(100)` 처럼 짧게 잡으면 → 회사 메일 (`firstname.lastname.long@subdomain.company.co.kr`) reject 사례. 한국 학생/기업 일부 도메인이 100 초과.
- `NULL` 허용하면 → 비밀번호 재설정 / 인증 메일 발송 불가한 user 가 발생. 검증 로직마다 null check 분기.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| `VARCHAR(320)` (RFC 5321 ABNF 의 path) | 실 사용 mailbox 의 maximum 은 254. 320 은 SMTP envelope 의 이론값 |
| `TEXT` | PostgreSQL 에선 같지만 다른 RDBMS / ORM 호환성, 인덱스 길이 제한 (MySQL ~767 byte) 우려 |
| `CITEXT` (case-insensitive type) | PostgreSQL 전용 + extension 필요. `lower(email)` 표현식 인덱스로 충분 |

**트레이드오프**
- Apple 의 "Hide My Email" / 소셜 가입 시 email 부재 케이스 → application 단에서 `{external_id}@apple-private.local` 같은 placeholder. nullable 로 풀면 알림 발송 코드 곳곳에 분기. [[../enums/social-provider-type]] 참고.

---

### 2.3 `phone VARCHAR(20)` + `phone_hash CHAR(64)`

**왜 필요한가 (phone)**
- 한국에선 휴대폰 본인인증 + 알림톡이 사실상 표준 (SMS / 카카오 알림톡).
- 정규화된 형식 `01012345678` 로 저장 — 검색 / 중복 체크 일관성.
- 20 char = 글로벌 대비 (`+82-10-1234-5678` 형식 + 안전 마진).

**왜 phone_hash (별도 컬럼)**
- DB 유출 시 phone 평문 = 개인정보보호법 위반 (한국 PIPC 과징금 대상).
- application 코드는 hash 로 검색 (`WHERE phone_hash = sha256(?)`) → 인덱스로 lookup.
- 평문은 알림톡 발송 직전에만 decrypt / fetch.

**안 하면 무슨 문제**
- phone 만 평문 + 인덱스 → DB 백업 유출 / SQL injection 시 전체 phone 평문 노출. 한국 사례: 야놀자 (2018), 인터파크 (2016) 등 phone 평문 노출 사고.
- phone_hash 없이 평문 검색 → 인덱스 lookup 은 가능하지만 `LIKE '%010-1234%'` 같은 검색은 풀스캔. 더 큰 문제는 보안.
- 정규화 안 하고 저장 → `010-1234-5678`, `01012345678`, `+821012345678` 모두 다른 row → 같은 사람 중복 가입 통과.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| `pgcrypto` 컬럼 암호화 (대칭) | 검색 시 매번 decrypt → 인덱스 X. 대량 조회 비용 폭증 |
| 분리 테이블 (phone_secrets) | 매 조회 join. 트랜잭션 / 동시성 복잡. 본 vault 규모엔 과잉 |
| hash 안 하고 RDS at-rest 만 | 디스크 도난 보호는 OK 지만 SQL injection / 백업 유출엔 무력 |

**트레이드오프**
- SHA-256 만으론 brute force 가능 (전화번호 공간 = 한국 휴대폰 약 5,000만 = rainbow table 가능).
- 강한 보호는 → HMAC-SHA256 + 서버 secret. 단 secret rotation 시 전체 user 의 hash 재계산 필요.
- 본 vault: SHA-256 (탐색 인덱스 + DB 백업 누출 대비) + RDS at-rest encryption. 강화는 [[encryption-at-rest]] 옵션.

---

### 2.4 `password_hash VARCHAR(255)` — nullable

**왜 필요한가**
- argon2 PHC 형식 (`$argon2id$v=19$m=65536,t=3,p=4$salt$hash`) ≈ 100~120 char.
- 255 = OWASP 권장 알고리즘 변경 대비 (`bcrypt` $2b$ ~60, `scrypt` ~100). 미래 알고리즘 교체 시 schema 변경 X.
- **소셜 가입** (provider_type=APPLE/GOOGLE/...) 은 password 없음 → nullable 필수.

**왜 NULL 허용 + CHECK 제약 조합**
```sql
CONSTRAINT chk_users_local_has_password
    CHECK (provider_type <> 'LOCAL' OR password_hash IS NOT NULL)
```
→ LOCAL 이면 NOT NULL 강제, 소셜이면 NULL 허용. application 만 의존 X — DB 가 마지막 방어선.

**안 하면 무슨 문제**
- `NOT NULL` 강제 → 소셜 가입 시 dummy hash 생성. 그 hash 가 brute force 되면 로그인 통과 가능 (소셜 유저가 password 로그인 우회).
- nullable 인데 CHECK 없으면 → LOCAL 가입인데 password_hash NULL 인 row 생성 가능 (application bug 시). 로그인 시 NullPointer 또는 빈 hash 매치 통과.
- `VARCHAR(100)` 으로 너무 좁히면 → argon2 m=128MB 강화 시 hash 길이 증가 → INSERT 실패. 운영 중 ALTER COLUMN.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| 별도 `credentials` table | LOCAL 만 row 있고 소셜은 row 없음 — 매 로그인 join. 작은 이득에 큰 복잡도 |
| bcrypt 만 사용 (`$2b$`) | argon2 가 GPU/ASIC 저항성 ↑. OWASP 2024 권장은 argon2id |
| 평문 저장 | 절대 불가. 법적 책임 + 기본 윤리 |

**트레이드오프**
- 알고리즘 마이그레이션 (bcrypt → argon2) 정책 필요. 본 vault: `needsRehash` 패턴 — 로그인 성공 시 옛 hash 면 새 알고리즘으로 재계산. [[../domain-model/password-hash-vo#6]].

---

### 2.5 `name VARCHAR(100) NOT NULL`

**왜 필요한가**
- 화면 표시 / 결제 / CS 응대.
- 100 = 한국어 ~33자 (UTF-8 3 byte) + 영문 풀네임 + 안전 마진.

**안 하면 무슨 문제**
- `VARCHAR(20)` 으로 좁히면 → 외국인 사용자 (`Maria José García-López de la Vega`) 잘림 / reject. 결제 명의 불일치 / 본인 인증 실패.
- `nullable` 면 → 영수증 / 결제 발급 시 NULL 표시. 환불 / 분쟁 시 본인 식별 불가.

**대안과 왜 안 됨**
- `first_name` + `last_name` 분리 → 한국/중국/일본은 single field 가 자연. i18n 정책 따라 결정. 본 vault: B2C 한국 우선 → single field.

**트레이드오프**
- 100 char 도 결국 짧음 — 진짜 풀네임 (인도 등) 은 더 김. RFC 6531 / vcard 표준 따르면 500+. B2C 한국 commerce 는 100 충분.
- Bean Validation `@Size(min=1, max=100)` + 도메인 VO 가 이중 방어. DB CHECK 까지 가면 과잉이라 안 거는 것이 본 vault 의 균형점.

---

### 2.6 `status VARCHAR(30) NOT NULL` + CHECK

**왜 필요한가**
- 회원의 생명주기 표현: 인증 대기 / 활성 / 정지 / 삭제.
- 각 상태마다 가능한 액션이 다름 (PENDING 은 로그인 불가, SUSPENDED 는 읽기만, ...).

**왜 VARCHAR + CHECK (not enum type)**
- PostgreSQL `CREATE TYPE ... AS ENUM` 은 값 추가 가능하지만 **삭제 / 순서 변경이 매우 어려움** (DROP + 모든 dependent column 변경).
- VARCHAR + CHECK 는 ALTER TABLE 으로 CHECK 만 갱신.
- 30 char = 가장 긴 값 `PENDING_VERIFICATION` (20) + 미래 추가 대비 (`PENDING_KYC_REVIEW` 같은).

**안 하면 무슨 문제**
- CHECK 없이 VARCHAR → 누군가 SQL 직접 INSERT 시 `'active'` (소문자) / `'ACTIVATED'` (오타) 가능. application 의 enum 매핑 실패 → 500 error 또는 silent ignore.
- `INT` 로 저장 → DB 만 보면 `status=2` 가 무슨 뜻인지 모름. 운영 시 SQL query 가독성 폭망.
- `BOOLEAN active` → 4가지 상태 표현 불가 (인증 대기 vs 정지 vs 삭제 구분 X).

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| `SMALLINT` (코드) | 가독성 ↓. status='1' 이 ACTIVE 인지 PENDING 인지 SQL 만 봐선 모름 |
| Native ENUM (PostgreSQL) | 값 변경 비용. 마이그레이션 시 매번 ALTER TYPE |
| 별도 status 테이블 + FK | 과잉. 매 조회 join. |

**트레이드오프**
- CHECK 제약 변경 시 (값 추가) → 마이그레이션 필요 (ALTER TABLE DROP CONSTRAINT + ADD). 작은 비용.
- 매핑은 JPA `@Enumerated(EnumType.STRING)` 필수 — ORDINAL 쓰면 enum 순서 변경 시 모든 row 의 의미가 깨짐 (재앙).

자세히: [[../enums/user-status]].

---

### 2.7 `role VARCHAR(20) NOT NULL DEFAULT 'USER'` + CHECK

**왜 필요한가**
- 권한 분리 (`USER`, `SELLER`, `ADMIN`, `MASTER`). Spring Security 의 `hasRole(...)` 매핑.
- DEFAULT 'USER' = 신규 가입자 자동 부여 (signup endpoint 에서 role 지정 X).

**안 하면 무슨 문제**
- DEFAULT 없으면 → signup INSERT 시 role 누락 → NOT NULL 위반. 매 호출에서 role 명시 강제.
- enum 외 값 통과 → Spring Security 의 `ROLE_???` 매핑 시 default deny 또는 권한 부여 실패.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| 별도 `user_roles` table (M:N) | 한 사용자가 여러 role 동시 보유 케이스 거의 없음 (USER+SELLER 도 SELLER 가 USER 포함하면 됨). 단순성 우선 |
| RBAC framework (Casbin 등) | 본 vault 규모엔 과잉. 권한 4종 + 계층 구조면 enum 충분 |

**트레이드오프**
- 권한 매트릭스가 복잡해지면 (5+ role × 50+ resource) — 별도 `permissions` 테이블 필요. 현재 정책: 4 role 까지는 enum.
- role 변경 시 즉시 반영 X (JWT 안에 박힌 role 이 access token 만료까지 유효). 정책: ADMIN 강등 시 강제 logout (revoke refresh + access blacklist).

자세히: [[../enums/role]].

---

### 2.8 `provider_type VARCHAR(20) NOT NULL DEFAULT 'LOCAL'` + CHECK

**왜 필요한가**
- 로그인 흐름 분기 (LOCAL = password 검증, APPLE/GOOGLE = OAuth IdP 검증).
- 보안 정책 분기 (LOCAL = password reset 메일, 소셜 = provider 의 reset 으로 안내).

**안 하면 무슨 문제**
- provider_type 없으면 → 같은 email 의 LOCAL/APPLE 구분 불가. APPLE 로 가입한 사용자에게 password reset 메일 발송 (revoke 없음 → 무한 reset 시도).
- (provider_type, external_id) UNIQUE 없으면 → 한 Apple 계정으로 여러 user row 생성 가능.

**대안과 왜 안 됨**
- 별도 `social_identities` 테이블 → 한 user 가 여러 provider 연동 (LOCAL+APPLE+GOOGLE) 시 깔끔. 단 본 vault 의 초기 단계: 한 user = 한 provider. 미래 확장 시 마이그레이션.

**트레이드오프**
- 같은 email 의 LOCAL/APPLE = 정책 결정 필요. 본 vault 의 default: **이메일 검증된 경우 자동 link, 아니면 reject** (UX 우선 사례) — 자세히 [[../design-decisions]].

자세히: [[../enums/social-provider-type]].

---

### 2.9 `external_id VARCHAR(100)` (소셜) + `apple_sub VARCHAR(100)`

**왜 필요한가 (external_id)**
- 소셜 provider 의 user 식별자 (Google `sub`, Apple `sub`, Kakao `id`).
- 우리 DB 의 `users.id` (ULID) 와 별개. 같은 사람이 provider 별로 다른 external_id.

**왜 apple_sub 따로**
- Apple 은 token revocation 시 `sub` 필요. external_id 로도 가능하지만 명시적 컬럼이 운영 디버깅 / 코드 가독성 ↑.
- Apple 의 "Sign in with Apple" 의 보안 요구 — sub 별도 추적 권장.

**안 하면 무슨 문제**
- external_id 없으면 → 소셜 가입 user 의 IdP 매핑 불가. 다음 로그인 시 같은 사람인지 판별 X → 매번 새 user row 생성.
- 100 char 보다 짧으면 → Google `sub` 가 보통 21 char 지만 Apple 의 일부 sub 는 더 김 (50+). reject 사례.

**대안과 왜 안 됨**
- Email 매칭으로 통합 → 같은 email 사용자가 LOCAL/APPLE/GOOGLE 모두 → 분리 안 됨. external_id 가 더 안전한 식별자.

**트레이드오프**
- external_id 가 NULL 인 row (LOCAL) 가 다수 → partial unique index (`WHERE external_id IS NOT NULL`) 로 처리.
- Apple 의 "Hide My Email" 의 random email → email UNIQUE 외에 (provider_type, external_id) UNIQUE 필요.

---

### 2.10 `email_verified_at TIMESTAMPTZ` / `phone_verified_at TIMESTAMPTZ`

**왜 필요한가 (boolean 대신 timestamp)**
- 단순 `bool email_verified` 보다 **언제 검증됐는지 audit** 가능.
- 비즈니스 규칙: "30일 이상 미인증 시 자동 휴면" 같은 정책에 timestamp 필요.
- 디버깅: CS 문의 "내 이메일 인증 언제 됐어요?" 응답.

**안 하면 무슨 문제**
- boolean 만 → 인증 후 30일 휴면 정책 불가. 별도 `email_verified_changed_at` 같은 컬럼 추가 (결국 timestamp).
- `NOT NULL DEFAULT false` boolean → 검증 안 한 user 가 false 로 가입. 같은 정보는 NULL 로도 표현 가능 (NULL = 미인증).

**대안과 왜 안 됨**
- 별도 `user_verifications` 테이블 → join 비용. 1:1 관계엔 같은 row 가 자연.
- email_verified BOOLEAN + email_verified_at TIMESTAMP → 둘 다 두면 정합성 (BOOLEAN=true 인데 timestamp=NULL?) 깨짐 위험.

**트레이드오프**
- NULL = false 와 의미 동일 → 코드에서 `emailVerifiedAt != null` 같은 표현 일관.
- 본 vault: timestamp 만. boolean view 가 필요하면 `(email_verified_at IS NOT NULL) AS email_verified`.

---

### 2.11 `version BIGINT NOT NULL DEFAULT 0` — `@Version`

**왜 필요한가**
- JPA 낙관 락. 동시 UPDATE 시 lost update 방지.
- 예: ADMIN 이 user.status 변경 + 사용자가 자기 name 변경 동시 → 한쪽이 다른 쪽 변경 덮어쓰기. version 으로 충돌 감지.

**안 하면 무슨 문제**
- 두 트랜잭션이 동시에 `SELECT user → 메모리 변경 → UPDATE` 패턴 → 마지막 commit 이 이긴다 (lost update).
- 실제 사례: 잔액 / 포인트 차감에서 흔한 버그. user 의 status 변경에서도 발생 가능.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| 비관 락 (`SELECT FOR UPDATE`) | row lock 점유 시간 ↑ → throughput ↓. user 변경 빈도 낮으면 낙관 락이 효율적 |
| 메모리 mutex / Redis lock | 분산 환경에서 동기화 비용. DB 자체로 가능한데 우회할 이유 X |

**트레이드오프**
- 충돌 시 OptimisticLockException → application 이 retry 또는 사용자에게 "다시 시도" 안내. retry 정책 명시.
- INSERT 시 version=0, UPDATE 시 자동 +1 — JPA 가 처리.

---

### 2.12 `created_at` / `updated_at` `TIMESTAMPTZ`

**왜 필요한가**
- 기본 audit. 분쟁 / CS / 분석.
- TIMESTAMPTZ = timezone-aware. UTC 저장 + 표시 시 사용자 timezone 변환.

**안 하면 무슨 문제**
- TIMESTAMP (without tz) → 서버 timezone 변경 시 의미가 바뀜. 다중 region 운영 시 재앙.
- updated_at 자동 갱신 안 하면 → 마지막 변경 시각 추적 불가. CS 응대 시 "언제 정보 바뀌었어요?" 답변 X.

**대안과 왜 안 됨**
- DB `now()` default + trigger 로 updated_at 자동 갱신 → 가장 robust. JPA `@PreUpdate` 보다 안전 (직접 SQL UPDATE 도 잡힘).
- 본 vault: application (`@CreatedDate`, `@LastModifiedDate`) + DB default. 이중 방어.

**트레이드오프**
- DB trigger 추가 시 → 마이그레이션 복잡 + 디버깅 어려움 (trigger 가 보이지 않게 컬럼 변경). application 단 명시 권장.

---

## 3. 인덱스

```sql
-- 1. lower(email) UNIQUE — 가장 중요
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

-- 2. (provider_type, external_id) UNIQUE — 소셜 가입 중복 차단
CREATE UNIQUE INDEX ux_users_provider_external
    ON users (provider_type, external_id)
    WHERE external_id IS NOT NULL;

-- 3. phone_hash UNIQUE — 휴대폰 검색 + 중복 차단 (옵션)
CREATE UNIQUE INDEX ux_users_phone_hash
    ON users (phone_hash)
    WHERE phone_hash IS NOT NULL;

-- 4. status 별 검색 (Admin)
CREATE INDEX ix_users_status_created
    ON users (status, created_at DESC)
    WHERE status <> 'DELETED';

-- 5. created_at 정렬 (Admin / Analytics)
CREATE INDEX ix_users_created_at ON users (created_at DESC);
```

### 3.1 왜 `lower(email)` 표현식 인덱스

**문제 상황**
```sql
-- ❌ 단순 unique
CREATE UNIQUE INDEX ix_users_email ON users (email);
```
- `Alice@X.com` 과 `alice@x.com` 이 둘 다 통과. **같은 사람이 두 계정 보유 가능**.
- 로그인 시 `WHERE email = ?` 가 case-sensitive → 사용자가 입력한 case 와 저장된 case 가 다르면 로그인 실패 (UX 재앙).

**해결**
```sql
-- ✅ 표현식 인덱스
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

-- 검색
SELECT * FROM users WHERE lower(email) = lower(?);
```
- DB 가 case-insensitive UNIQUE 보장. application 의 `toLowerCase` 누락 버그도 DB 가 막음.

**왜 application `toLowerCase` 만으로 부족**
- application 코드는 변경됨 → 옛 코드는 toLowerCase 했지만 새 코드는 안 할 수 있음.
- 다른 클라이언트 (admin 스크립트, batch) 가 직접 INSERT 시 우회.
- DB 가 마지막 진실의 원천.

**대안과 트레이드오프**
- `CITEXT` (PostgreSQL extension) → 컬럼 자체가 case-insensitive. 깔끔하지만 extension 의존 + MySQL 등 호환 X.
- 모두 lowercase 저장 + 일반 UNIQUE → application 단의 변환 누락에 취약. 본 vault 의 권장은 **저장은 lowercase (Service 변환) + 인덱스는 lower(email) (DB 안전망)**.

---

### 3.2 왜 partial unique `(provider_type, external_id) WHERE external_id IS NOT NULL`

**문제 상황**
- LOCAL user 는 external_id=NULL. NULL 끼리는 NULL 끼리는 UNIQUE 위반이 아니라서 (PostgreSQL/표준 SQL 의 NULL 처리) 무한 INSERT 가능.
- 단순 UNIQUE (LOCAL row 가 99% 일 경우) → 인덱스 크기 폭증 + tree 가 NULL 로 차서 효율 ↓.

**해결**
```sql
CREATE UNIQUE INDEX ux_users_provider_external
    ON users (provider_type, external_id)
    WHERE external_id IS NOT NULL;
```
- 소셜 user 만 인덱스에 들어감. LOCAL (NULL external_id) 은 제외.
- 인덱스 크기 ↓ + 의미적으로 정확 (LOCAL 은 external_id 로 중복 식별 X).

**안 하면 무슨 문제**
- 한 Apple 계정으로 여러 user row 생성 가능 → 로그인 시 어떤 row 가 진짜?
- 인덱스 크기 비효율 (1000만 user 중 100만이 소셜이면 1000만 row 다 인덱싱).

---

### 3.3 왜 partial index `WHERE status <> 'DELETED'`

**문제 상황**
- soft delete 정책 → DELETED row 가 누적. 5년 후 누적 user 의 ~30% 가 DELETED 가능.
- Admin 검색 (`WHERE status='ACTIVE'`) 시 DELETED row 도 인덱스 스캔 — 비효율.

**해결**
```sql
CREATE INDEX ix_users_status_created
    ON users (status, created_at DESC)
    WHERE status <> 'DELETED';
```
- 인덱스 크기 ↓ (DELETED 제외).
- ACTIVE/SUSPENDED 검색 시 더 작은 인덱스 활용 → I/O ↓.

**트레이드오프**
- DELETED user 의 검색 (CS 환불 등) 은 풀스캔 또는 별도 인덱스 필요. 빈도 낮으니 OK.

---

### 3.4 인덱스 미사용 모니터링

```sql
-- pg_stat_user_indexes 활용
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public' AND relname = 'users'
ORDER BY idx_scan;
```

**왜 모니터링**
- 인덱스 추가는 쉽지만 제거는 위험 (운영 중 누가 의존하는지 모름). `idx_scan = 0` 가 수개월 = 안전한 제거 후보.
- 인덱스마다 INSERT/UPDATE 시 추가 비용. 무용 인덱스 정리 = throughput 개선.

---

## 4. 조회 패턴

| 패턴 | SQL | 빈도 | 인덱스 | 응답 시간 목표 |
| --- | --- | --- | --- | --- |
| 로그인 (이메일) | `WHERE lower(email) = lower(?)` | 매우 잦음 | `ux_users_email` | p99 < 10ms |
| 로그인 (휴대폰) | `WHERE phone_hash = sha256(?)` | 잦음 (한국) | `ux_users_phone_hash` | p99 < 10ms |
| 소셜 로그인 | `WHERE provider_type=? AND external_id=?` | 잦음 | `ux_users_provider_external` | p99 < 10ms |
| 사용자 정보 (`/me`) | `WHERE id = ?` | 매 인증 요청 | PK | p99 < 5ms |
| Admin 검색 | `WHERE name LIKE ? AND status IN (?)` | 드뭄 | (pg_trgm 옵션) | p99 < 500ms |
| Admin 목록 | `WHERE status = ? ORDER BY created_at DESC` | 드뭄 | `ix_users_status_created` | p99 < 200ms |

> 💡 응답 시간 목표는 인덱스 미스 / cache miss 여부 따라. 본 vault 가정: shared_buffers 충분 + warm cache.

---

## 5. 정합성

### 5.1 무엇을 보장 — 책임 계층

| 보장 | 어디서 | 왜 그 계층 |
| --- | --- | --- |
| 이메일 case-insensitive UNIQUE | DB (`ux_users_email`) | application 변환 누락 / 다른 클라이언트 우회 방어 |
| 소셜 (provider, external_id) UNIQUE | DB (`ux_users_provider_external`) | race condition 시 application 의 existsBy 검사가 부족 |
| status / role / provider 유효값 | DB CHECK + JPA `EnumType.STRING` | enum 외 값이 DB 들어가면 application load 시 매핑 실패 |
| LOCAL → password_hash NOT NULL | DB CHECK `chk_users_local_has_password` | application bug 시 inconsistent row 생성 막음 |
| 도메인 무결성 (email 형식, name 길이) | Domain VO + Bean Validation | DB 까지 보내기 전 reject (성능 + UX) |

**왜 3중 (Bean Validation + Domain VO + DB constraint)**
- Bean Validation: HTTP 요청 단계의 1차 reject. 짧고 빠름.
- Domain VO: 도메인 layer 에 들어오는 모든 값 검증 (REST 외 batch/CLI 도 보호).
- DB constraint: 마지막 방어선. application bug / direct SQL 도 막음.
- **한 곳만 의존하면 누군가 우회 시 무방비**. 다층 방어 = 운영 안정성.

### 5.2 race condition — DB 가 진실의 원천

```
A: existsByEmail('a@x') → false       B: existsByEmail('a@x') → false
A: INSERT                              B: INSERT (UNIQUE 위반!)
A: 성공                                 B: DataIntegrityViolationException
                                       → EmailAlreadyExistsException → 409
```

**왜 application `existsBy` 만 의존 X**
- 두 트랜잭션 사이의 race window 가 ms 단위지만 throughput 높으면 자주 발생.
- 분산 lock (Redis) 으로 막을 수 있지만 인프라 부담 + DB UNIQUE 가 충분.

자세히: [[../transactions#4 동시성]].

---

## 6. Soft Delete 정책

### 6.1 왜 hard delete X

```sql
DELETE FROM users WHERE id = ?    -- ❌
```

**문제**
- FK CASCADE 위험 — `orders`, `payments` 가 user_id 참조. user 삭제 시 모든 주문 / 결제 기록 삭제 = 회계 / 분쟁 자료 손실.
- 법적 의무 — 한국 전자상거래법 5년 (계약/결제 기록 보존), 통신비밀보호법 (수사 협조).
- audit trail 손상 — "탈퇴한 user 가 무슨 행동 했었는지" 추적 불가.

### 6.2 soft delete + email anonymize

```sql
UPDATE users SET
    email = 'deleted-' || id || '@deleted.example.com',     -- email UNIQUE 회피
    phone = NULL,
    phone_hash = NULL,
    password_hash = NULL,
    status = 'DELETED',
    updated_at = now()
WHERE id = ?;
```

**왜 이 방식**
- email = `deleted-{id}@deleted.example.com` → UNIQUE 인덱스에서 자연스럽게 unique (id 가 ULID 라 충돌 X).
- 옛 user 의 이메일로 재가입 가능 (`alice@x.com` 이 탈퇴 후 재가입 — UNIQUE 위반 X).
- PII (phone, phone_hash, password_hash) 는 NULL → 개인정보보호법의 "탈퇴 후 즉시 파기" 요건 충족.
- 단 user.id, created_at, 주문 이력은 그대로 → 회계 / 분쟁 대응 가능.

**안 하면 무슨 문제**
- email 그대로 두면 → 재가입 시 UNIQUE 위반. 사용자 UX 망가짐.
- PII 그대로 두면 → 개인정보보호법 위반 (탈퇴 후 보유 사유 없는데 보관).

### 6.3 partial unique index 옵션 (대안)

```sql
DROP INDEX ux_users_email;
CREATE UNIQUE INDEX ux_users_email_active
    ON users (lower(email))
    WHERE status <> 'DELETED';
```

**왜 대안인가**
- email 안 바꾸고 그대로 보존 → 분석 / audit 에서 옛 user 의 email 활용 가능.
- DELETED row 의 email 은 UNIQUE 제약 X → 재가입 가능.

**왜 본 vault 가 anonymize 선택**
- 단순성 — partial index 는 DELETED row 의 email 이 남아 PII 보관 정당화 어려움.
- 법적 안전 — anonymize 가 개인정보보호법 해석상 더 명확.

### 6.4 GDPR / 개인정보보호법 — 30일 후 완전 파기 옵션

```sql
-- 30일 지난 DELETED user 의 모든 PII 파기 + audit 핵심만 남김
UPDATE users SET
    email = NULL,
    name = '[deleted]',
    -- phone, phone_hash, password_hash 는 이미 NULL
    -- id, created_at, status 만 보존
WHERE status = 'DELETED' AND updated_at < now() - INTERVAL '30 days';
```

**왜**
- 한국 개인정보보호법 / GDPR — "보유 기간 만료 후 즉시 파기" 의무.
- 다만 회계 / 결제 기록 보존 의무 (5년) 와 충돌 → `id` 만 남기고 PII 제거 = 균형점.

---

## 7. 마이그레이션 — V1

```sql
-- src/main/resources/db/migration/V1__create_users.sql
CREATE TABLE users (
    -- 위 schema
);

-- Indexes
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));
CREATE UNIQUE INDEX ux_users_provider_external
    ON users (provider_type, external_id) WHERE external_id IS NOT NULL;
CREATE INDEX ix_users_status_created ON users (status, created_at DESC)
    WHERE status <> 'DELETED';
CREATE INDEX ix_users_created_at ON users (created_at DESC);
```

**왜 V1 에 인덱스 함께**
- 운영 중 인덱스 추가 = `CONCURRENTLY` 필요 (락 회피). 초기 = 같은 마이그레이션에 포함.
- Flyway 의 V1 은 어떤 환경이든 동일 결과 보장.

자세히: [[migrations]].

---

## 8. JPA Entity 매핑

```java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id @Column(length = 26) private String id;

    @Column(nullable = false, length = 254) private String email;
    @Column(length = 20) private String phone;
    @Column(name = "phone_hash", length = 64) private String phoneHash;
    @Column(name = "password_hash", length = 255) private String passwordHash;
    @Column(nullable = false, length = 100) private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30) private UserStatus status;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private Role role;

    @Enumerated(EnumType.STRING)
    @Column(name = "provider_type", nullable = false, length = 20)
    private SocialProviderType providerType;

    @Column(name = "external_id", length = 100) private String externalId;
    @Column(name = "apple_sub", length = 100) private String appleSub;

    @Column(name = "email_verified_at") private Instant emailVerifiedAt;
    @Column(name = "phone_verified_at") private Instant phoneVerifiedAt;

    @Version @Column(nullable = false) private long version;

    @Column(name = "created_at", nullable = false, updatable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;
}
```

> 💡 **왜 `EnumType.STRING`** — ORDINAL 사용 시 enum 순서 바꾸면 모든 row 의 의미가 깨짐. 예: USER=0, SELLER=1 → 나중에 alphabetical sort 로 코드 정리 → ADMIN=0, MASTER=1, SELLER=2, USER=3 → 기존 row 의 0 (USER) 이 ADMIN 으로 해석되는 재앙. STRING 은 절대 안 깨짐.
>
> 💡 **왜 `updatable = false` 의 `created_at`** — INSERT 후 변경 불가. application bug 로 created_at 이 바뀌는 사고 (audit 손상) 방지.
>
> 💡 **왜 `@Version`** — 동시 수정 충돌 감지. 단 retry 정책 명시 필요 — 그냥 500 error 던지면 UX 망가짐.

---

## 9. 함정 모음 — "이걸 안 하면 X 가 터짐"

### 함정 1 — `email` 만 UNIQUE (lower 없이)
`alice@x.com` 과 `Alice@X.com` 둘 다 통과. **실제 사례**: 사용자가 첫 가입 시 대문자 입력 → 다음 로그인 시 소문자 입력 → "계정 없음" 에러 → 다시 가입 → 같은 사람의 두 계정. CS 폭증.
→ `lower(email)` 표현식 인덱스 필수.

### 함정 2 — Soft delete 후 재가입 막힘
`ux_users_email` 가 `lower(email)` UNIQUE → DELETED user 의 email 도 잡음. 같은 email 로 재가입 시 409.
→ email anonymize 또는 partial index (`WHERE status <> 'DELETED'`).

### 함정 3 — `password_hash NOT NULL`
소셜 user 는 password 없음. NOT NULL 강제 시 dummy hash 생성 — brute force 로 로그인 우회 가능.
→ nullable + CHECK 제약 (`provider_type='LOCAL' → password_hash NOT NULL`).

### 함정 4 — `phone_hash` 없이 평문 검색
운영 중 DB 백업 유출 시 phone 평문 전체 노출 → PIPC 과징금. (실제 한국 사례 多).
→ phone_hash 저장 + 검색은 hash 로.

### 함정 5 — `@Enumerated(EnumType.ORDINAL)`
enum 순서 (`USER, SELLER, ADMIN`) 가 alphabetical 로 자동 reorder 되면 `0=USER` 가 `0=ADMIN` 으로. 모든 user row 의 권한이 침묵 변경.
→ 반드시 `EnumType.STRING`.

### 함정 6 — `created_at` 의 default 누락
JPA 가 안 set + DB default 도 없으면 NULL. 정렬 / 분석 / audit 망가짐.
→ DB DEFAULT now() + JPA `@CreatedDate` 이중.

### 함정 7 — `updated_at` 자동 갱신 누락
직접 SQL UPDATE 시 updated_at 안 바뀜 → 마지막 변경 시각 추적 X.
→ application (`@PreUpdate`) + 중요 변경은 별도 audit log.

### 함정 8 — Soft delete 후 audit log 없음
"누가 / 왜 삭제했는지" 추적 X. CS 분쟁 시 입증 불가.
→ 별도 `user_audit_log` 또는 `deleted_by` / `delete_reason` 컬럼.

### 함정 9 — `@Version` 없이 동시 수정
ADMIN 이 status=SUSPENDED 변경 + 사용자가 name 변경 동시 → 마지막이 이김. lost update.
→ `@Version` 필수 + 충돌 시 retry 정책.

### 함정 10 — Admin 검색용 인덱스 누락
status 별 검색이 full scan → 1000만 user 에서 1초+. Admin 대시보드 망함.
→ partial index `ix_users_status_created`.

### 함정 11 — VARCHAR(255) email 무지성
RFC 5321 path = 254 max. 255 로 잡아도 운영엔 무방하지만 — 254 가 명시적 / 의미 있음.
→ `VARCHAR(254)`.

### 함정 12 — `external_id` UNIQUE 누락
한 Apple 계정으로 여러 user row 생성 가능 → 로그인 시 어떤 row?
→ partial unique `(provider_type, external_id) WHERE external_id IS NOT NULL`.

### 함정 13 — Apple 의 sub 추적 누락
Apple revocation webhook 처리 시 sub 로 user 찾아야 → `apple_sub` 별도 컬럼.

### 함정 14 — `name` 영문 제한 / 짧음
외국인 사용자 풀네임 reject. 결제 명의 불일치 → 환불 불가.
→ `VARCHAR(100)` + 도메인 검증.

### 함정 15 — Boolean `email_verified`
"30일 미인증 휴면" 정책 불가. 별도 timestamp 필요해짐.
→ 처음부터 `TIMESTAMPTZ` 만.

---

## 10. 관련

- [[database|↑ database hub]]
- [[../domain-model/user-aggregate]] — User 의 도메인 메서드
- [[../enums/user-status]] / [[../enums/role]] / [[../enums/social-provider-type]]
- [[id-strategy]] — ULID 의 깊은 비교
- [[encryption-at-rest]] — PII 보호
- [[../security]] — auth 전반 보안
