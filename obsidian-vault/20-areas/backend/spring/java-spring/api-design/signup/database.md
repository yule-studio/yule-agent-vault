---
title: "signup §4 — DB 선택 / 스키마 / 조회 패턴 / 정합성"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - database
---

# signup §4 — DB 선택 / 스키마 / 조회 패턴 / 정합성

**[[signup|↑ signup hub]]**  ·  ← [[architecture]]  ·  → [[security]]

---

## 1. DB 선택 — 왜 RDB (PostgreSQL)

| 후보 | 적합 | 이유 |
| --- | --- | --- |
| **PostgreSQL / MySQL** | ✅ | `UNIQUE (lower(email))` 한 줄로 동시 가입 race 차단. ACID 트랜잭션. |
| MongoDB | ⚠️ | unique index 가능. 동시성 미묘 + 분산 환경 까다로움. |
| DynamoDB | ⚠️ | conditional put 가능. 단 query 패턴 PK 의존적. |
| Redis | ❌ | 영속 / 백업 / 관계가 핵심인 도메인엔 부적합. |

**결정**: PostgreSQL 16.

이유:
- email unique 동시성 보장 = DB constraint 한 줄 (=사실상 공짜)
- 추후 `orders`, `payments` 등 강한 관계 도메인과 JOIN
- backup / PITR 표준
- 한국 SaaS production 표준

---

## 2. 스키마

### 2.1 users 테이블

```sql
-- src/main/resources/db/migration/V1__create_users.sql
CREATE TABLE users (
    id              CHAR(26) PRIMARY KEY,                  -- ULID
    email           VARCHAR(254) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,                 -- argon2id PHC 문자열
    name            VARCHAR(100) NOT NULL,
    status          VARCHAR(30)  NOT NULL,
    version         BIGINT       NOT NULL DEFAULT 0,        -- @Version 낙관 락
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- 핵심: case-insensitive unique
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

-- DELETED 제외한 status 별 검색
CREATE INDEX ix_users_status ON users (status) WHERE status <> 'DELETED';

-- 생성일 정렬 (admin 화면)
CREATE INDEX ix_users_created_at ON users (created_at DESC);
```

### 2.2 약관 동의 history (필수)

```sql
-- V2__create_user_terms.sql
CREATE TABLE terms (
    id            CHAR(26) PRIMARY KEY,
    code          VARCHAR(50) NOT NULL,           -- "service-terms" / "privacy-policy"
    version       VARCHAR(20) NOT NULL,            -- "v1.0", "v1.1"
    title         VARCHAR(200) NOT NULL,
    content       TEXT NOT NULL,
    required      BOOLEAN NOT NULL,
    effective_at  TIMESTAMPTZ NOT NULL,
    use_yn        VARCHAR(1) NOT NULL DEFAULT 'Y',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_terms_code_version ON terms (code, version);
CREATE INDEX ix_terms_active ON terms (code, effective_at DESC) WHERE use_yn = 'Y';

CREATE TABLE user_terms_consent_history (
    id          CHAR(26) PRIMARY KEY,
    user_id     CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    terms_id    CHAR(26) NOT NULL REFERENCES terms(id),
    consent_yn  VARCHAR(1) NOT NULL,                -- 'Y' / 'N'
    agreed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_terms_consent_user ON user_terms_consent_history (user_id);
CREATE INDEX ix_terms_consent_terms ON user_terms_consent_history (terms_id);
```

> **함정**: 약관 동의를 `users` 테이블의 `marketing_agreed BOOLEAN` 으로 두면 — 약관 버전 변경 / 재동의 / 철회 history 추적 불가. **별도 history 테이블 필수** (한국 개인정보 정책).

### 2.3 (선택) 이메일 outbox

```sql
-- V3__create_email_outbox.sql
CREATE TABLE email_outbox (
    id          CHAR(26) PRIMARY KEY,
    to_email    VARCHAR(254) NOT NULL,
    template    VARCHAR(50) NOT NULL,                -- "verification" / "welcome"
    payload     JSONB NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts    INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at     TIMESTAMPTZ
);
CREATE INDEX ix_email_outbox_pending ON email_outbox (status, created_at)
    WHERE status = 'PENDING';
```

별도 워커 ([[../email-verification]]) 가 polling.

### 2.4 (선택) Idempotency keys

```sql
CREATE TABLE idempotency_keys (
    key          VARCHAR(50) PRIMARY KEY,
    resource_id  CHAR(26) NOT NULL,
    user_email   VARCHAR(254),
    response     JSONB,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX ix_idempotency_expires ON idempotency_keys (expires_at);
```

---

## 3. 조회 패턴 (이 도메인의 read 사용처)

| 패턴 | SQL | 빈도 | 인덱스 |
| --- | --- | --- | --- |
| 로그인 — 이메일로 user lookup | `WHERE lower(email) = ?` | 매우 잦음 | `ux_users_email` |
| 가입 1차 검증 — 중복 체크 | `SELECT EXISTS WHERE lower(email) = ?` | 모든 signup | 같음 |
| `/me` — ID 로 lookup | `WHERE id = ?` | 매 인증 요청 | PK |
| Admin 검색 — name LIKE | `WHERE name ILIKE '%x%'` | 드뭄 | (필요 시 trgm) |
| Admin 목록 — status / 가입일 | `WHERE status = ? ORDER BY created_at DESC` | 드뭄 | `ix_users_status`, `ix_users_created_at` |
| 약관 동의 조회 | `WHERE user_id = ?` | 가입 직후 + admin | `ix_terms_consent_user` |

→ **이메일 lookup 이 절대 다수**. case-insensitive unique 가 핵심.

---

## 4. 정합성 기준

### 4.1 어떤 보장이 필요한가

| 보장 | 누가 책임 |
| --- | --- |
| 이메일 case-insensitive unique | **DB** (`ux_users_email`) — 진실의 원천 |
| password_hash 가 argon2 형식 | **Domain** (`PasswordHash` value object) |
| status 가 enum 값 | **DB CHECK** + JPA `EnumType.STRING` |
| terms_id 존재 | **DB FK** |
| user 삭제 시 약관 동의도 cascade | **DB ON DELETE CASCADE** (또는 soft delete) |
| name 1~100 자 | **Domain** + DB `VARCHAR(100)` |
| 동시 가입 race 차단 | **DB UNIQUE** (application 1차 검증은 사용자 친화 메시지 용도) |

### 4.2 Soft delete vs Hard delete

| 영역 | 정책 |
| --- | --- |
| `users` | **Soft delete** — `status = 'DELETED'`. 행 자체 보존 (참조 무결성 위해). 이메일은 anonymize 가능 (`alice@x.com` → `deleted-01HZ@deleted.example.com`). |
| `user_terms_consent_history` | **Hard delete** (CASCADE) — 사용자가 GDPR 데이터 삭제권 행사 시. 단 회계 / 분쟁 대비라면 보존. |
| `email_outbox` | TTL (30일 후 삭제). |

> **함정 (DELETED 의 이메일 재가입)**: `alice@x.com` 으로 가입 후 DELETED → `alice@x.com` 으로 재가입? `ux_users_email` UNIQUE 가 그대로 막음. **anonymize 또는 UNIQUE 에 `WHERE status <> 'DELETED'` partial index** 정책 선택.

```sql
-- 옵션 A — anonymize on delete
UPDATE users SET email = 'deleted-' || id || '@deleted.example.com',
                  status = 'DELETED'
WHERE id = ?;

-- 옵션 B — partial unique index
DROP INDEX ux_users_email;
CREATE UNIQUE INDEX ux_users_email_active
    ON users (lower(email)) WHERE status <> 'DELETED';
```

본 레시피: **옵션 A** (단순 + 추후 분쟁 대비 audit).

### 4.3 동시성 — 같은 이메일 race

```
Thread A:                          Thread B:
  existsByEmail('a@x') → false      existsByEmail('a@x') → false
  User.register(...)                User.register(...)
  users.save(user_A)                users.save(user_B)
       ↓                                 ↓
  INSERT 성공                          INSERT 실패 (UNIQUE 위반)
                                          ↓
                                     DataIntegrityViolationException
                                          ↓
                                     EmailAlreadyExistsException → 409
```

→ **1차 application 검증은 친절한 에러용**, **DB UNIQUE 가 진실**.

---

## 5. ID 전략 — 왜 ULID

| 후보 | 장점 | 단점 |
| --- | --- | --- |
| `BIGSERIAL` | 단순 | enumeration 위험 (`/users/1, /users/2, ...`) |
| `UUIDv4` | 추측 불가 | 정렬 X / B-tree 분산 → 쓰기 비용 |
| **ULID** | 시간순 정렬 + 추측 불가 | 26 char string |
| UUIDv7 | ULID 동등 + 표준 | 라이브러리 의존 |

**선택**: **ULID** — app-side 발급 (`IdGenerator`), DB 가 sequence 모름.

```java
@Bean public IdGenerator idGenerator() {
    return () -> ULID.random();        // io.azam.ulidj
}
```

---

## 6. 마이그레이션 / Flyway

```
src/main/resources/db/migration/
├── V1__create_users.sql
├── V2__create_user_terms.sql
├── V3__create_email_outbox.sql
└── V4__add_users_version.sql            (예 — 추후 @Version 추가 시)
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate              # 운영 절대 update / create-drop X
  flyway:
    enabled: true
    locations: classpath:db/migration
```

→ **Flyway 가 진실의 원천**. JPA 는 mapping 만, 스키마는 만들지 않음.

---

## 7. 관련

- [[signup|↑ signup hub]]
- [[architecture]] — 이전 (§3)
- [[security]] — 다음 (§5)
- [[../../../../database/postgresql/security|↗ PG 보안]]
- [[../../database/jpa#11.5.1]] · [[../../database/mybatis#10.5.1]]
