---
title: "PostgreSQL 트랜잭션 / MVCC / Isolation"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:30:00+09:00
tags:
  - database
  - postgresql
  - transaction
  - mvcc
  - isolation
---

# PostgreSQL 트랜잭션 / MVCC / Isolation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ACID / Isolation / MVCC / VACUUM |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. ACID

| 속성 | 의미 | 보장 방법 |
| --- | --- | --- |
| **A**tomicity | 전부 / 전무 | WAL + ROLLBACK |
| **C**onsistency | 제약 유지 | Constraints, FK, CHECK |
| **I**solation | 트랜잭션 격리 | MVCC + Lock |
| **D**urability | 커밋 후 영속 | WAL + fsync |

---

## 2. BEGIN / COMMIT / ROLLBACK

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 실패 시
BEGIN;
  ...
ROLLBACK;
```

### 2.1 SAVEPOINT

```sql
BEGIN;
  INSERT INTO orders (...);
  SAVEPOINT s1;
  INSERT INTO items (...);   -- 실패할 수도
EXCEPTION WHEN OTHERS THEN
  ROLLBACK TO s1;
END;
COMMIT;
```

### 2.2 Autocommit
기본 — 모든 statement 가 자체 트랜잭션. `BEGIN` 없이 실행하면 즉시 COMMIT.

---

## 3. Isolation Level

| 레벨 | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | 가능 (이론) | 가능 | 가능 | 가능 |
| **READ COMMITTED** (기본) | ❌ | 가능 | 가능 | 가능 |
| **REPEATABLE READ** | ❌ | ❌ | ❌ (PG 는 막음) | 가능 |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | ❌ |

PostgreSQL 은 READ UNCOMMITTED 도 내부적으로 READ COMMITTED 로 동작.

### 3.1 설정

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 또는 트랜잭션 시작 시
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- 세션 기본
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 3.2 각 레벨의 의미

#### READ COMMITTED (기본)
- 매 statement 가 자신의 snapshot 을 봄
- 같은 트랜잭션 내 두 SELECT 가 다른 결과 가능
- 대부분의 OLTP 에서 충분

#### REPEATABLE READ
- 트랜잭션 시작 시점의 snapshot 을 끝까지
- 같은 SELECT 는 같은 결과
- Write skew (서로 다른 행을 읽고 쓰는 경합) 가능

#### SERIALIZABLE (SSI)
- 직렬화 가능 — 마치 모든 트랜잭션이 순차 실행한 것과 같은 결과
- PG 는 **SSI (Serializable Snapshot Isolation)** 로 구현 — lock 없음
- 충돌 시 `serialization_failure` (40001) — 클라이언트가 재시도

```python
# 재시도 패턴
for attempt in range(5):
    try:
        with conn.transaction(isolation_level="SERIALIZABLE"):
            ...
        break
    except SerializationFailure:
        continue
```

---

## 4. MVCC (Multi-Version Concurrency Control)

핵심: **읽기는 쓰기를 막지 않고, 쓰기는 읽기를 막지 않는다**.

### 4.1 동작

```
UPDATE 시:
  1. 기존 row 의 xmax = 현재 TXID
  2. 새 row 삽입, xmin = 현재 TXID
  → 같은 row 의 2 버전 공존
  
다른 트랜잭션 (snapshot xid = 100) 이 읽을 때:
  - xmin <= 100 < xmax  → 보임 (자기 시점의 버전)
  - xmin > 100         → 안 보임 (미래)
  - xmax < 100         → 안 보임 (이미 지워진 과거)
```

### 4.2 xmin / xmax

```sql
SELECT xmin, xmax, * FROM users WHERE id = 1;
-- xmin: 이 row 를 만든 TXID
-- xmax: 이 row 를 지운 TXID (0 이면 살아있음)
```

### 4.3 결과
- 락 없는 동시 읽기
- 일관된 snapshot
- 단점: **dead tuple** (오래된 버전) 누적 → VACUUM 필요

---

## 5. VACUUM — Dead tuple 정리

### 5.1 왜 필요한가
MVCC 의 부산물 — UPDATE / DELETE 한 옛 버전은 디스크에 남아있음 → **bloat**.

### 5.2 VACUUM 종류

```sql
VACUUM users;                -- 일반 — 공간 재사용 표시
VACUUM (ANALYZE) users;      -- + 통계 갱신
VACUUM FULL users;           -- 테이블 재작성 (ACCESS EXCLUSIVE lock!)
VACUUM (VERBOSE, ANALYZE) users;
```

| 종류 | lock | 디스크 회수 | 실무 |
| --- | --- | --- | --- |
| VACUUM | 짧은 lock | ❌ (재사용 표시만) | 일상 |
| VACUUM FULL | ACCESS EXCLUSIVE | ✅ | 비상시만 |
| `pg_repack` (extension) | 짧은 lock | ✅ | 운영 권장 |

### 5.3 autovacuum

```ini
autovacuum = on
autovacuum_vacuum_scale_factor = 0.1   # 10% 변경 시
autovacuum_analyze_scale_factor = 0.05 # 5% 변경 시 ANALYZE
```

### 5.4 진단

```sql
SELECT
  relname,
  n_live_tup, n_dead_tup,
  ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
  last_vacuum, last_autovacuum,
  last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_pct DESC NULLS LAST;
```

### 5.5 Transaction ID Wraparound
TXID 는 32 bit — 약 20 억 트랜잭션 후 wrap. VACUUM 이 oldest xmin 까지 정리해야 안전. wraparound 임박 시 강제 정지 → **VACUUM 은 필수**.

---

## 6. Lock

### 6.1 행 락 (Row Lock)

```sql
-- 명시적 행 락
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;       -- 독점
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE; -- 약함
SELECT * FROM accounts WHERE id = 1 FOR SHARE;         -- 공유
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;     -- 가장 약함
```

### 6.2 SKIP LOCKED — 큐 패턴

```sql
SELECT id, payload FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- 다른 워커가 잠근 행은 건너뜀 — 작업 큐 구현
```

### 6.3 NOWAIT

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- 락 못 잡으면 즉시 에러
```

### 6.4 테이블 락

| 락 모드 | 의미 |
| --- | --- |
| `ACCESS SHARE` | SELECT |
| `ROW SHARE` | SELECT FOR UPDATE |
| `ROW EXCLUSIVE` | INSERT/UPDATE/DELETE |
| `SHARE` | CREATE INDEX (without CONCURRENTLY) |
| `SHARE UPDATE EXCLUSIVE` | VACUUM, ANALYZE |
| `EXCLUSIVE` | 거의 모두 막음 |
| `ACCESS EXCLUSIVE` | 완전 단독 (DROP, TRUNCATE, VACUUM FULL) |

### 6.5 Lock 진단

```sql
SELECT
  bl.pid AS blocked_pid,
  a.query AS blocked_query,
  bl_act.usename AS blocked_user,
  kl.pid AS blocking_pid,
  ka.query AS blocking_query,
  kl_act.usename AS blocking_user
FROM pg_locks bl
JOIN pg_stat_activity bl_act ON bl_act.pid = bl.pid
JOIN pg_locks kl ON kl.transactionid = bl.transactionid AND kl.pid != bl.pid
JOIN pg_stat_activity kl_act ON kl_act.pid = kl.pid
WHERE NOT bl.granted;
```

---

## 7. Deadlock

PG 는 자동 감지 → 한 쪽 트랜잭션 abort + `deadlock_detected` 에러.

```sql
SHOW deadlock_timeout;   -- 기본 1s
```

### 7.1 회피

- 항상 같은 순서로 lock (`id ASC`)
- 트랜잭션 짧게
- 인덱스로 lock 영역 축소

---

## 8. 트랜잭션 + 에러

```sql
BEGIN;
  INSERT INTO t VALUES (1);
  INSERT INTO t VALUES ('bad');  -- 에러
  INSERT INTO t VALUES (2);      -- ❌ 무시됨 (transaction aborted)
COMMIT;                          -- = ROLLBACK
```

→ 트랜잭션 중 에러는 **전체 abort**. SAVEPOINT 로 부분 회복.

### 8.1 ON_ERROR_ROLLBACK (psql)
psql 의 자동 SAVEPOINT — `\set ON_ERROR_ROLLBACK on`.

---

## 9. 두 단계 커밋 (2PC)

```sql
BEGIN;
  ...
PREPARE TRANSACTION 'tx-abc';

-- 다른 시점에
COMMIT PREPARED 'tx-abc';
-- 또는
ROLLBACK PREPARED 'tx-abc';
```

분산 트랜잭션 / XA 용. 일반 앱은 거의 안 씀.

---

## 10. Read-only 트랜잭션

```sql
BEGIN READ ONLY;
SELECT ...;
COMMIT;
```

또는 `default_transaction_read_only = on` 으로 standby 강제.

---

## 11. 함정

### 함정 1 — Long-running transaction
오래 열린 트랜잭션 → VACUUM 못 함 → bloat 폭증. `idle_in_transaction_session_timeout` 설정.

### 함정 2 — autocommit 의존 + 다중 statement
ORM 의 autocommit 가정. 일관성 필요한 곳은 명시적 transaction.

### 함정 3 — SELECT 후 UPDATE (Read-Modify-Write)
다른 트랜잭션과 race. `SELECT ... FOR UPDATE` 또는 `UPDATE ... WHERE col = old_value` 패턴.

### 함정 4 — Phantom read (REPEATABLE READ 이상에서는 막힘)
같은 조건 SELECT 가 새 행을 본다. PG 는 REPEATABLE READ 부터 막음.

### 함정 5 — SSI 의 재시도 누락
SERIALIZABLE 사용 시 `40001` 재시도 로직 필수.

### 함정 6 — DDL 도 트랜잭션 안에서
대부분의 DDL 은 트랜잭션. `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY` 는 예외 — 트랜잭션 밖에서.

### 함정 7 — VACUUM 못 도는 환경
`oldest xmin` 이 멀면 VACUUM 효과 없음. `pg_stat_activity` 에서 long-running 확인.

---

## 12. 학습 자료

- PostgreSQL Documentation Ch. 13 (Concurrency)
- **PostgreSQL 14 Internals** — Egor Rogov (필독)
- **PostgreSQL High Performance** — Hans-Jürgen Schönig

---

## 13. 관련

- [[explain-analyze]] — 락 진단
- [[performance-tuning]] — autovacuum 튜닝
- [[postgresql]] — PostgreSQL hub
