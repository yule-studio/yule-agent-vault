---
title: "auth §5 — DB 스키마 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - hub
---

# auth §5 — DB 스키마 (Hub)

**[[../signup|↑ signup hub]]**  ·  ← [[../design-decisions|design-decisions]]  ·  → [[../domain-model/domain-model|domain-model]]

> auth 도메인의 모든 table / 인덱스 / 제약 / 정합성 정책.
> 각 table 이 자기 노트 + 공통 정책 (ID / 마이그레이션 / 암호화).

---

## 1. 이 폴더의 노트

### 1.1 Table 별

| 노트 | Table | 목적 |
| --- | --- | --- |
| [[users-table]] | `users` | 사용자 본체 + soft delete + lower(email) UNIQUE |
| [[terms-tables]] | `terms`, `user_terms_consent_history` | 약관 + 동의 history |
| [[refresh-tokens-table]] | `refresh_tokens` | JWT refresh (rotation chain) |
| [[verification-tokens-table]] | `email_verification_tokens`, `phone_verifications`, `password_reset_tokens` | 3종 공통 패턴 |
| [[email-outbox-table]] | `email_outbox` | outbox 패턴 + worker |

### 1.2 정책 / 횡단

| 노트 | 무엇 |
| --- | --- |
| [[id-strategy]] | ULID 결정 + `CHAR(26)` 통일 |
| [[migrations]] | Flyway 운영 + 마이그레이션 정책 + 롤백 |
| [[encryption-at-rest]] | RDS encryption / column-level 암호화 / hash 인덱스 |

---

## 2. 전체 ERD

```mermaid
erDiagram
    users ||--o{ user_terms_consent_history : "user_id"
    terms ||--o{ user_terms_consent_history : "terms_id"
    users ||--o{ refresh_tokens : "user_id"
    users ||--o{ email_verification_tokens : "user_id"
    users ||--o{ password_reset_tokens : "user_id"
    users }o--o{ email_outbox : "to_email (FK X)"

    users {
        char(26) id PK "ULID"
        varchar(254) email UK "lower(email) UNIQUE"
        varchar(20) phone
        char(64) phone_hash UK "SHA-256"
        varchar(255) password_hash "argon2id PHC, nullable (social)"
        varchar(30) status "PENDING/ACTIVE/SUSPENDED/DELETED"
        varchar(20) role "USER/SELLER/ADMIN/MASTER"
        varchar(20) provider_type "LOCAL/APPLE/GOOGLE/KAKAO/NAVER"
        varchar(100) external_id "(provider, external_id) UNIQUE"
        bigint version "@Version 낙관 락"
        timestamptz created_at
    }

    terms {
        char(26) id PK
        varchar(50) code "(code, version) UNIQUE"
        varchar(20) version
        boolean required
        timestamptz effective_at
        varchar(1) use_yn
    }

    user_terms_consent_history {
        char(26) id PK
        char(26) user_id FK
        char(26) terms_id FK
        varchar(1) consent_yn "Y/N"
        timestamptz agreed_at
        varchar(45) ip_address
    }

    refresh_tokens {
        char(26) id PK "jti"
        char(26) user_id FK
        char(64) token_hash UK "SHA-256(raw)"
        varchar(20) status "ACTIVE/ROTATED/REVOKED/EXPIRED"
        char(26) rotated_to_id "chain 추적"
        timestamptz expires_at
    }

    email_verification_tokens {
        char(26) id PK
        char(26) user_id FK
        char(64) token_hash UK
        varchar(20) status "ACTIVE/USED/REVOKED/EXPIRED"
        timestamptz expires_at "TTL 24h"
    }

    password_reset_tokens {
        char(26) id PK
        char(26) user_id FK
        char(64) token_hash UK
        varchar(20) status
        timestamptz expires_at "TTL 30m"
    }

    email_outbox {
        char(26) id PK
        varchar(254) to_email "user FK X (시점 의존)"
        varchar(50) template
        jsonb payload
        varchar(20) status "PENDING/PROCESSING/SENT/FAILED/DEAD_LETTER"
        timestamptz next_attempt_at
    }
```

### 2.1 관계 요약

```mermaid
flowchart LR
    users[users<br/>auth 의 핵심]
    terms[terms<br/>버전 관리]
    consent[user_terms_consent_history<br/>동의 audit]
    rt[refresh_tokens<br/>JWT refresh + rotation]
    verify[email_verification_tokens<br/>password_reset_tokens]
    phone[phone_verifications<br/>Redis 우선]
    outbox[email_outbox<br/>PENDING → SENT]

    users -->|user_id FK<br/>ON DELETE CASCADE| consent
    terms -->|terms_id FK| consent
    users -->|user_id FK<br/>CASCADE| rt
    users -->|user_id FK<br/>CASCADE| verify
    users -.->|phone 식별<br/>FK X| phone
    users -.->|to_email<br/>FK X 시점 의존| outbox

    style users fill:#fef3c7
    style outbox fill:#dbeafe
    style phone fill:#dbeafe
```

> 💡 **FK 있음 (실선)** = ON DELETE CASCADE — user 삭제 시 같이.
> 💡 **FK 없음 (점선)** = phone 은 가입 전 존재 / outbox 는 시점 의존 (email 변경 후에도 옛 발송 OK).

---

## 3. 공통 컨벤션

### 3.1 ID — ULID `CHAR(26)`

모든 PK 가 ULID. 자세히: [[id-strategy]].

### 3.2 시간 — `TIMESTAMPTZ` (UTC)

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

→ PG 가 UTC 저장 + 응답 시 클라 timezone 변환.

### 3.3 enum — `VARCHAR + CHECK`

```sql
status VARCHAR(30) NOT NULL CHECK (status IN ('PENDING_VERIFICATION', 'ACTIVE', ...))
```

→ JPA `@Enumerated(EnumType.STRING)` 와 일치.

### 3.4 soft delete vs hard delete

| Table | 정책 |
| --- | --- |
| `users` | **soft delete** (`status='DELETED'` + email anonymize) |
| `user_terms_consent_history` | hard delete (CASCADE) — GDPR 시 |
| `refresh_tokens` / `*_verification_tokens` | hard delete (7일 후 cleanup) |
| `email_outbox` | hard delete (30일 후 cleanup) |

### 3.5 indexing 원칙

- **PK** — 자동 인덱스
- **UNIQUE (lookup)** — `lower(email)`, `token_hash`
- **Foreign key** — 자동 안 만들어짐 (PG). 명시:
  ```sql
  CREATE INDEX ix_user_terms_consent_user ON user_terms_consent_history (user_id);
  ```
- **Partial index** — DELETED 제외:
  ```sql
  CREATE INDEX ix_users_active ON users (created_at DESC) WHERE status <> 'DELETED';
  ```
- **인덱스 = INSERT 비용** — 무한 추가 X. `pg_stat_user_indexes` 모니터.

자세히: [[../../database/postgresql/security|↗ PG 보안]] · [[../../database/jpa#11.5.1 signup 의 JPA]].

---

## 4. 트랜잭션 / 정합성

### 4.1 한 트랜잭션 안의 INSERT 들

가입 흐름:
```sql
BEGIN;
  INSERT INTO users (...);                          -- 1
  INSERT INTO user_terms_consent_history (...);     -- N (약관 수)
COMMIT;
-- AFTER_COMMIT 후 별도 트랜잭션
  INSERT INTO email_outbox (...);
```

[[../transactions]] 의 트랜잭션 경계 결정.

### 4.2 Race condition 의 진실의 원천

| 시나리오 | 1차 방어 | 진실의 원천 |
| --- | --- | --- |
| email 중복 가입 | `existsByEmail` (application) | `ux_users_email` UNIQUE |
| 같은 refresh 동시 사용 | hash lookup | `ux_refresh_tokens_hash` UNIQUE |
| 토큰 중복 발급 | application 검증 | `ux_*_tokens_hash` UNIQUE |

→ **DB constraint 가 마지막 안전망**.

---

## 5. 검색 패턴 — 어떤 쿼리가 자주

| 쿼리 | 빈도 | 인덱스 |
| --- | --- | --- |
| `users WHERE lower(email) = ?` (로그인) | 매우 잦음 | `ux_users_email` |
| `users WHERE id = ?` | 매 인증 요청 | PK |
| `refresh_tokens WHERE token_hash = ?` | 토큰 갱신 | `ux_refresh_tokens_hash` |
| `email_outbox WHERE status = 'PENDING'` | 워커 polling | `ix_email_outbox_pending` (partial) |
| `users WHERE phone_hash = ?` (휴대폰 login) | 자주 | `ux_users_phone_hash` (옵션) |

자세히: [[users-table#3 조회 패턴]].

---

## 6. 마이그레이션 — Flyway 한 곳에서

```
src/main/resources/db/migration/
├── V1__create_users.sql                    [[users-table]]
├── V2__create_terms_and_consent.sql        [[terms-tables]]
├── V3__create_refresh_tokens.sql           [[refresh-tokens-table]]
├── V4__create_email_verification_tokens.sql [[verification-tokens-table]]
├── V5__create_phone_verifications.sql      [[verification-tokens-table]]
├── V6__create_password_reset_tokens.sql    [[verification-tokens-table]]
├── V7__create_email_outbox.sql             [[email-outbox-table]]
└── V8__add_users_phone_columns.sql         (예 — 후속 migration)
```

자세히: [[migrations]].

---

## 7. 관련

- [[../signup|↑ signup hub]]
- [[../domain-model/domain-model|↗ 도메인 모델]] — DB ↔ 도메인 매핑
- [[../../database/database|↗ database hub]] — JPA / MyBatis / 공존
- [[../../database/postgresql/security|↗ PG 보안]]
- [[../../database/jpa#11.5.1]] — Adapter 적용
