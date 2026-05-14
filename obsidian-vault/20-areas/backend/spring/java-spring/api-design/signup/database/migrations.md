---
title: "Flyway 운영 + 마이그레이션 정책"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - migrations
  - flyway
---

# Flyway 운영 + 마이그레이션 정책

**[[database|↑ database hub]]**

> 스키마 변경은 모두 Flyway. **수동 ALTER 금지**.
> 이 정책이 없으면 **환경 간 schema drift, 롤백 불가, 운영 사고 시 복구 불가**.

---

## 1. 왜 Flyway (Liquibase / 수동 SQL 아님)

### 1.1 왜 마이그레이션 도구 필수

**수동 SQL 의 문제 — 구체적 사고**

| 시나리오 | 결과 |
| --- | --- |
| dev DB 에 `ALTER TABLE users ADD COLUMN ...` 수동 실행 | staging / prod 에 적용 누락 → 환경 간 schema drift |
| 운영자가 prod 만 ALTER, dev 안 함 | 새 코드 배포 시 dev 에서 작동 X, "내 PC 에선 됐는데" |
| 변경 이력 미기록 | "이 컬럼 언제 왜 추가됐어?" 답 X. 회귀 시 복구 어려움 |
| 두 개발자가 같은 schema 변경 PR | merge 후 어느 쪽이 먼저 실행되는지 모름 |

**Flyway 의 해결**
- `db/migration/V{N}__{name}.sql` — 버전 / 순서 명시.
- `flyway_schema_history` 테이블 — 어떤 마이그레이션이 언제 적용됐는지 audit.
- 모든 환경에서 같은 순서로 같은 SQL 실행 보장.

### 1.2 Flyway vs Liquibase

| 항목 | Flyway | Liquibase |
| --- | --- | --- |
| 형식 | 순수 SQL | XML / YAML / JSON / SQL |
| 학습 곡선 | 낮음 | 높음 |
| 롤백 | 별도 V로 forward 권장 | undo changeset 지원 |
| 변경 detect | checksum | tag / context |
| 운영 인지도 (한국 SaaS) | ✅ 표준 | 점유율 낮음 |

**왜 Flyway 선택 (본 vault)**
- SQL 그대로 — DBA / 운영자가 보는 코드와 일치.
- 학습 곡선 낮음 — 신규 합류자도 며칠 안에 적응.
- 한국 Spring Boot 생태계에서 사실상 표준.

**Liquibase 가 맞는 경우**
- DB 종류가 여러 개 (PostgreSQL + MySQL + Oracle) — 같은 changeset 으로 다른 SQL 생성.
- 강한 형식 검증 / 자동 롤백 (undo) 필요.

---

## 2. 디렉토리 구조

```
src/main/resources/db/migration/
├── V1__create_users.sql                       — users 테이블
├── V2__create_terms_and_consent.sql           — terms + consent_history
├── V2_1__seed_initial_terms.sql               — 초기 약관 데이터
├── V3__create_refresh_tokens.sql              — refresh_tokens
├── V4__create_email_verification_tokens.sql   — email 인증
├── V5__create_phone_verifications.sql         — phone 인증
├── V6__create_password_reset_tokens.sql       — password reset
├── V7__create_email_outbox.sql                — outbox
└── V8__add_users_phone_columns.sql            — (후속 schema 변경 예시)
```

### 2.1 명명 규칙 — `V{N}__{description}.sql`

**왜 이 형식**
- `V1` / `V2` 같은 정수 — 적용 순서 명확.
- `__` (이중 underscore) — 버전과 description 구분.
- description 은 snake_case — 운영 시 grep 용이.

**왜 V1.1 / V2.3 같은 minor 사용**
- `V2_1__seed_initial_terms.sql` 같은 보조 마이그레이션 — V2 후 V3 전 실행.
- 시드 데이터 / hotfix 같이 같은 도메인의 작은 변경.

**안 하면 무슨 문제 (date prefix `V20260115__...`)**
- 두 개발자가 같은 날 V20260115 만들면 충돌.
- `V{N}` 가 충돌 감지 명확.

---

## 3. 마이그레이션 작성 규칙

### 3.1 한 번 적용된 V 는 변경 X

**왜**
- 이미 prod 적용된 V1 변경 시 → Flyway checksum mismatch → 운영 중단.
- 변경이 필요하면 **새 V** 작성 (forward-only).

**안 하면 무슨 문제**
- "V1 의 컬럼 길이 100 → 200 변경하려고 V1 수정" → staging 정상 (clean DB), prod 에서 checksum 불일치 → 배포 멈춤.
- 강제 repair 시 → 변경 사항이 prod 에 반영 안 됨 (V1 은 이미 실행됨, 변경된 내용 무시).

**올바른 흐름**
```sql
-- V8__alter_users_name_length.sql
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(200);
```

### 3.2 idempotent — 가능하면

```sql
-- ✅ 안전
CREATE TABLE IF NOT EXISTS users (...);
CREATE INDEX IF NOT EXISTS ix_users_email ON users (...);

-- ❌ 재실행 시 실패
CREATE TABLE users (...);
```

**왜 idempotent**
- Flyway 가 기본적으로 같은 V 재실행 안 함. 단 schema_history 손상 / 강제 repair 시 안전.
- 운영 시 디버깅 용 재실행 가능.

**언제 idempotent 어려운가**
- ALTER TABLE ADD COLUMN — `IF NOT EXISTS` PostgreSQL 9.6+ 만.
- 데이터 마이그레이션 (UPDATE) — `WHERE col IS NULL` 같은 조건.

### 3.3 DDL 와 DML 분리

```sql
-- ❌ 한 마이그레이션에 DDL + 큰 DML
V8__add_status_and_backfill.sql
ALTER TABLE users ADD COLUMN status VARCHAR(30);
UPDATE users SET status = 'ACTIVE';   -- 1000만 row → 30분 락
```

**왜 분리**
- 큰 UPDATE 는 락 점유 → 운영 중단.
- 분리하면 ALTER (수 초) + 별도 batch UPDATE (점진적).

**올바른 흐름**
```sql
-- V8__add_status_column.sql (수 초)
ALTER TABLE users ADD COLUMN status VARCHAR(30);

-- 별도 application batch (점진적)
-- 1000 row 씩 UPDATE WHERE status IS NULL LIMIT 1000;
```

### 3.4 인덱스 — `CONCURRENTLY` (운영 중 추가 시)

```sql
-- 운영 중 추가
CREATE INDEX CONCURRENTLY ix_users_status ON users (status);
```

**왜 CONCURRENTLY**
- 일반 `CREATE INDEX` 는 ACCESS EXCLUSIVE 락 → INSERT/UPDATE 멈춤.
- CONCURRENTLY 는 락 안 잡지만 시간 ↑ (full scan 2회).

**주의**
- Flyway 가 트랜잭션 안에 실행 시 CONCURRENTLY 작동 X.
- Flyway config: `outOfOrder=true` + 별도 `Vxxx_concurrent_index.sql` 에 `-- $$nontransactional$$` 헤더 (Flyway 9+).

---

## 4. 환경별 운영

### 4.1 환경

| 환경 | 적용 시점 | 자동/수동 |
| --- | --- | --- |
| dev (로컬) | application 시작 시 자동 | 자동 (`flyway.enabled=true`) |
| dev (공용) | CI 가 main merge 시 | 자동 |
| staging | release branch 배포 시 | 자동 |
| prod | release 배포 시 (사람 승인) | 수동 트리거 또는 자동 (정책) |

### 4.2 왜 환경별 분리

**dev 자동의 이유**
- 개발자가 매번 수동 적용 시 누락 / 헷갈림.
- 시작 시 자동 적용 = 즉시 코드 작성 가능.

**prod 수동 / 승인 이유**
- 실수 시 데이터 손상 / 복구 불가.
- 큰 변경 (큰 ALTER / 데이터 마이그레이션) 은 배포 시간 ↑ → 운영 슬랏 결정 필요.
- DBA 검토 / 승인 필수.

**안 하면 무슨 문제**
- prod 자동 → 실수 SQL 이 즉시 적용 → 데이터 손상.
- 수동만 → 운영자가 누락 → schema drift.

---

## 5. 롤백 정책 — Forward-only

### 5.1 왜 forward-only

**전통적 롤백 (undo)**
- V8 적용 후 문제 시 → "rollback V8" → V8 의 reverse SQL 실행.
- 문제: 데이터 손실 (V8 이 컬럼 추가 후 데이터 INSERT 되면 reverse 시 데이터 사라짐).

**forward-only**
- V8 의 문제 발견 → V9 작성 (V8 의 변경을 다시 되돌리는 SQL).
- 데이터 손실 X (필요시 V9 안에서 데이터 마이그레이션).

**구체적 예시**

```sql
-- V8__add_email_alias_column.sql (잘못 추가)
ALTER TABLE users ADD COLUMN email_alias VARCHAR(254);

-- V9__remove_email_alias_column.sql (forward-fix)
ALTER TABLE users DROP COLUMN email_alias;
```

### 5.2 언제 진짜 롤백 (DB restore) 가 필요한가

- V8 이 prod 적용 후 → 30분 사이 데이터 손상 → V9 으로 fix 불가.
- DB snapshot 으로 restore (RDS automated backup, point-in-time recovery).
- 응급 상황 — DBA 와 사전 합의된 절차 필요.

---

## 6. 큰 테이블 변경 전략

### 6.1 ALTER TABLE 의 락 종류 (PostgreSQL)

| 변경 | 락 | 영향 |
| --- | --- | --- |
| ADD COLUMN (without default) | ACCESS EXCLUSIVE (수 ms) | 거의 무해 (PG 11+) |
| ADD COLUMN (with default) | ACCESS EXCLUSIVE + 전체 row 재작성 | 큰 테이블 = 분/시간 |
| ALTER TYPE | ACCESS EXCLUSIVE + 데이터 변환 | 큰 테이블 = 시간+ |
| DROP COLUMN | ACCESS EXCLUSIVE (수 ms) | 데이터는 안 사라짐 (재구성 시 사라짐) |
| CREATE INDEX | ACCESS EXCLUSIVE | INSERT/UPDATE 멈춤 |
| CREATE INDEX CONCURRENTLY | SHARE UPDATE EXCLUSIVE | 운영 가능 |

### 6.2 안전한 큰 ALTER 패턴 — "expand/contract"

**예시**: users.name 길이 100 → 200

```sql
-- V8__widen_users_name (수 ms)
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(200);
```

→ PostgreSQL 12+ 는 VARCHAR length 증가는 메타데이터만 변경. 즉시 완료.

**예시**: users.email 컬럼 분리 (email + email_local + email_domain)

1. **Expand** — 새 컬럼 추가 (즉시)
```sql
ALTER TABLE users ADD COLUMN email_local VARCHAR(64);
ALTER TABLE users ADD COLUMN email_domain VARCHAR(255);
```

2. **Backfill** — 점진적 데이터 채우기 (application batch)
```sql
UPDATE users SET
  email_local = split_part(email, '@', 1),
  email_domain = split_part(email, '@', 2)
WHERE email_local IS NULL AND id IN (SELECT id FROM users WHERE email_local IS NULL LIMIT 1000);
```

3. **Application 변경** — 새 컬럼 사용 + dual-write.

4. **Contract** — 옛 email 컬럼 제거 (모든 reader 가 새 컬럼 사용 확인 후)
```sql
ALTER TABLE users DROP COLUMN email;
```

**왜 expand/contract**
- 운영 중 무중단 변경 가능.
- 단계마다 롤백 가능 (각 단계가 작은 변경).
- 큰 락 회피.

---

## 7. 시드 데이터 (initial data)

```sql
-- V2_1__seed_initial_terms.sql
INSERT INTO terms (id, code, version, title, content, required, effective_at, use_yn)
VALUES (...);
```

### 7.1 왜 Flyway 로 (application bootstrap 아님)

**Application bootstrap 의 문제**
- 다중 인스턴스 → race condition. 두 인스턴스가 동시 INSERT 시도.
- 시작 시점에 DB schema 가 준비됐는지 보장 X.
- 운영자가 "어떻게 들어간 데이터?" 추적 어려움.

**Flyway 시드 의 장점**
- 단일 실행 보장 (V 가 적용된 후 재실행 X).
- audit — 어떤 V 에서 어떤 데이터 들어왔는지 명확.

### 7.2 환경별 시드

```
db/migration/         — 공통 시드 (모든 환경)
db/migration_dev/     — dev 전용 (테스트 user 등)
db/migration_test/    — test 전용
```

**application.yml**
```yaml
spring:
  flyway:
    locations:
      - classpath:db/migration
      - classpath:db/migration_${spring.profiles.active}
```

---

## 8. Flyway 운영 명령

```bash
# 적용 가능한 마이그레이션 미리보기
./mvnw flyway:info

# 모두 적용
./mvnw flyway:migrate

# 적용 안 된 V 가 있어도 강제 (재해 복구)
./mvnw flyway:repair

# 베이스라인 — 이미 있는 DB 에 Flyway 도입
./mvnw flyway:baseline
```

### 8.1 왜 `flyway:repair` 위험

- schema_history 의 checksum 강제 갱신.
- 잘못 사용 시 실제 schema 와 history 가 어긋남 → 다음 마이그레이션 잘못된 곳에서 시작.
- DBA / TechLead 만 사용 + audit log.

### 8.2 왜 `flyway:baseline` 필요

- 운영 중인 DB 에 Flyway 처음 도입 시 — 현재 schema 를 V0 으로 인정.
- baseline 없이 V1 부터 실행 시도 → "이미 있는 테이블" 에러.

---

## 9. CI 검증

### 9.1 마이그레이션 자동 테스트

```yaml
# .github/workflows/ci.yml
- name: Flyway dry-run on Testcontainers
  run: ./mvnw test -Dtest=*MigrationTest
```

**왜**
- PR 단계에서 마이그레이션이 깨끗한 DB 에서 끝까지 실행되는지 확인.
- 의존성 누락 (V8 이 V7 의 컬럼 의존하는데 V7 누락 등) 사전 감지.

### 9.2 schema 비교 (옵션)

- prod 의 schema 와 코드 base 의 schema 가 동기화 됐는지 정기 확인.
- `pg_dump --schema-only` + diff.

---

## 10. JPA `ddl-auto` 와의 관계

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate    # 운영
    # 또는 create-drop (test)
```

**왜 `validate` 권장**
- Flyway 가 schema 의 진실의 원천.
- JPA 는 entity ↔ schema 매핑 검증만.

**왜 `update` / `create` 금지 (prod)**
- application 이 schema 변경 → audit X, Flyway 와 충돌.
- 운영 시 dangerous.

**언제 `create-drop` 가 맞는가**
- 통합 테스트 (Testcontainers). 매 테스트마다 깨끗한 DB.

---

## 11. 함정 모음 — "이걸 안 하면 X 가 터짐"

### 함정 1 — 적용된 V 의 SQL 수정
checksum mismatch → 배포 멈춤. forward-only 원칙 위반.
→ 새 V 작성.

### 함정 2 — 큰 ALTER 를 한 V 에
운영 중 1시간 락 → 서비스 중단.
→ expand/contract 패턴.

### 함정 3 — CONCURRENTLY 없이 인덱스 추가
INSERT/UPDATE 멈춤 → 사용자 영향.
→ 운영 환경은 CONCURRENTLY + non-transactional.

### 함정 4 — 데이터 마이그레이션을 마이그레이션 안에
1000만 row UPDATE → DB 멈춤.
→ 별도 application batch + 점진적.

### 함정 5 — schema_history 손상 후 무리한 repair
실제 schema 와 history 어긋남 → 다음 마이그레이션 어디서 시작할지 모름.
→ DBA 와 절차.

### 함정 6 — 환경별 시드 데이터 한 마이그레이션
prod 에 dev 의 테스트 user 들어감. 보안 사고.
→ profile 별 location.

### 함정 7 — prod 자동 마이그레이션
실수 SQL 이 즉시 prod 적용. 데이터 손상.
→ prod 는 수동 / 사람 승인.

### 함정 8 — JPA `ddl-auto: update` (prod)
application 이 schema 변경. audit / Flyway 와 충돌.
→ `validate` 만.

### 함정 9 — 마이그레이션 자체에 테스트 없음
PR 단계에서 의존성 / 순서 문제 미감지 → prod 배포 시 fail.
→ Testcontainers 기반 마이그레이션 테스트.

### 함정 10 — V 번호 충돌 (두 PR 같은 번호)
머지 시 어느 쪽 먼저? 헷갈림.
→ PR 머지 직전 V 번호 점검 / CI lint.

### 함정 11 — DDL 안에 transaction-unsafe 명령
`CREATE INDEX CONCURRENTLY` 가 트랜잭션 안 작동 X. Flyway 의 자동 트랜잭션 분리 설정 필요.

### 함정 12 — 롤백 시 새 마이그레이션 (V9 reverse) 잊고 V8 SQL 수정
checksum mismatch 재발.
→ forward-only 절대 원칙.

---

## 12. 관련

- [[database|↑ database hub]]
- [[users-table]] · [[terms-tables]] · [[refresh-tokens-table]] · [[verification-tokens-table]] · [[email-outbox-table]] — 각 V 의 대상
- [[../operations]] — 배포 / 운영 정책
- [[../../database/jpa#7 ddl-auto]] — JPA 와의 관계
