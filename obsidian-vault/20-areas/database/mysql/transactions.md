---
title: "MySQL 트랜잭션 / Isolation / 락 / Gap Lock"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:45:00+09:00
tags:
  - database
  - mysql
  - transaction
  - lock
---

# MySQL 트랜잭션 / Isolation / 락

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ACID / Isolation / Gap Lock / Next-Key |

**[[mysql|↑ MySQL hub]]**

---

## 1. ACID

InnoDB 만 ACID. 트랜잭션은 `BEGIN/COMMIT/ROLLBACK`.

```sql
START TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
SET autocommit = 0;           -- 매 statement autocommit 끔
SET autocommit = 1;           -- 기본
```

⚠️ DDL (`ALTER`, `CREATE`, `DROP`) 은 **자동 COMMIT** — 트랜잭션 안에서 롤백 불가.

---

## 2. Isolation Level

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SHOW VARIABLES LIKE 'transaction_isolation';
```

| 레벨 | Dirty | Non-Repeatable | Phantom | 락 동작 |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | 가능 | 가능 | 가능 | 거의 X |
| READ COMMITTED | ❌ | 가능 | 가능 | Record Lock 만 |
| **REPEATABLE READ** (기본) | ❌ | ❌ | **❌ (Next-Key)** | Next-Key Lock |
| SERIALIZABLE | ❌ | ❌ | ❌ | 모든 SELECT 공유 락 |

MySQL InnoDB 의 기본은 **REPEATABLE READ** — Next-Key Lock 으로 Phantom 도 방지.

### 2.1 REPEATABLE READ 의 특수성
PostgreSQL 의 REPEATABLE READ 는 phantom 가능 (snapshot 시점 고정).
MySQL 은 Next-Key Lock 으로 phantom 도 방지 → 더 강함 (대신 락 ↑).

---

## 3. MVCC

InnoDB 의 MVCC:
- 각 행에 `DB_TRX_ID` (마지막 변경 트랜잭션) + `DB_ROLL_PTR` (undo log 포인터)
- READ COMMITTED — 매 SELECT 가 snapshot 새로
- REPEATABLE READ — 첫 SELECT 의 snapshot 끝까지
- 다른 트랜잭션의 변경은 undo log 추적

---

## 4. 락 종류

### 4.1 Record Lock
인덱스 레코드 자체.

### 4.2 Gap Lock
인덱스 사이 빈 공간. **새 INSERT 막음** (phantom 방지).

```
인덱스: 10, 20, 30
SELECT ... WHERE id BETWEEN 15 AND 25 FOR UPDATE;
→ Gap Lock: (10, 20], (20, 30]
→ 16~24 INSERT 불가
```

### 4.3 Next-Key Lock
Record + Gap (REPEATABLE READ 기본).

### 4.4 Insert Intention Lock
INSERT 시 의도 표시 — gap lock 과 호환 (gap 에 새 행 INSERT).

### 4.5 AUTO-INC Lock
AUTO_INCREMENT 직렬화.

```ini
innodb_autoinc_lock_mode = 2   # 기본 (binlog ROW)
```

| 모드 | 의미 |
| --- | --- |
| 0 | 옛 — 긴 lock |
| 1 | 일부만 lock — INSERT...SELECT 등 |
| 2 (기본) | 가장 약함 — interleaved |

---

## 5. SELECT 와 락

```sql
SELECT ...                       -- 비잠금 (snapshot)
SELECT ... FOR UPDATE             -- 독점 락
SELECT ... FOR SHARE              -- 공유 락 (8.0+)
SELECT ... LOCK IN SHARE MODE     -- 옛 표기
SELECT ... FOR UPDATE NOWAIT      -- 즉시 에러
SELECT ... FOR UPDATE SKIP LOCKED -- 잠긴 건 건너뜀 (큐!)
```

### 5.1 SKIP LOCKED — 작업 큐

```sql
START TRANSACTION;
SELECT id FROM jobs WHERE status='pending'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs SET status='processing' WHERE id = ?;
COMMIT;
```

---

## 6. SAVEPOINT

```sql
START TRANSACTION;
INSERT INTO orders (...) VALUES (...);
SAVEPOINT before_items;
INSERT INTO items (...) VALUES (...);
ROLLBACK TO SAVEPOINT before_items;   -- items 만 취소
COMMIT;
```

---

## 7. Deadlock

자동 감지. 한쪽 트랜잭션 abort + `Deadlock found` (1213).

```sql
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 섹션
```

### 7.1 회피
- 항상 같은 순서로 락 (`id ASC`)
- 트랜잭션 짧게
- 인덱스로 락 범위 좁힘
- 재시도 로직

```python
for attempt in range(5):
    try:
        with conn.begin():
            ...
        break
    except DeadlockError:
        time.sleep(0.01 * 2 ** attempt)
```

---

## 8. 진단

### 8.1 활성 트랜잭션

```sql
SELECT * FROM information_schema.INNODB_TRX;
```

### 8.2 락 대기

```sql
-- 8.0+
SELECT * FROM performance_schema.data_lock_waits;
SELECT * FROM performance_schema.data_locks;

-- sys 뷰 (가장 편함)
SELECT * FROM sys.innodb_lock_waits;
```

### 8.3 길게 도는 트랜잭션 종료

```sql
SHOW PROCESSLIST;
KILL 12345;
KILL QUERY 12345;   -- 트랜잭션은 살리고 쿼리만
```

---

## 9. binlog 와 트랜잭션

```ini
binlog_format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
```

binlog + redo log 의 **그룹 커밋** — XA two-phase 로 일관성.
`sync_binlog=1` + `flush_log=1` = 안전, 약간 느림. 운영 표준.

---

## 10. 1 commit / N statement

```sql
START TRANSACTION;
-- 1
INSERT INTO logs VALUES (...);
INSERT INTO logs VALUES (...);
-- ...
INSERT INTO logs VALUES (...);   -- 1000 개
COMMIT;
```

→ commit 1 회만 fsync. 대량 적재 시 효과 큼 (단, 너무 길면 undo / lock 누적).

---

## 11. 트랜잭션 + 예외

InnoDB 는 statement 실패 시 자동 롤백 X — 전체 커밋 / 롤백 결정은 앱.

```sql
-- ❌ 위험: insert 실패해도 commit 됨
START TRANSACTION;
INSERT ...;        -- 에러
INSERT ...;        -- 정상
COMMIT;
```

→ 앱에서 첫 에러에 ROLLBACK.

---

## 12. SQL_SAFE_UPDATES

```sql
SET SQL_SAFE_UPDATES = 1;
DELETE FROM users;   -- 에러 — WHERE 없음
```

운영 클라이언트에 활성 권장 (실수 방지).

---

## 13. XA Transaction (분산)

```sql
XA START 'tx1';
... 다중 DB ...
XA END 'tx1';
XA PREPARE 'tx1';
XA COMMIT 'tx1';
```

거의 안 씀 (성능 ↓). 일반 앱은 saga pattern.

---

## 14. 함정

### 함정 1 — DDL 의 자동 COMMIT
`ALTER` 실행 = 이전 트랜잭션 끝. 마이그레이션 도구는 별도 트랜잭션 관리.

### 함정 2 — REPEATABLE READ 의 락 폭증
`SELECT ... FOR UPDATE` 가 Next-Key Lock 으로 큰 범위 잠금. 인덱스 없으면 테이블 전체.

### 함정 3 — `READ COMMITTED` 전환의 함의
Phantom 가능. 일부 워크로드에 OK (락 ↓), 신중히.

### 함정 4 — Long Transaction
undo 누적 → 다른 read 도 느려짐. `idle_transaction_timeout` (8.0+).

### 함정 5 — `autocommit = 0` 잊음
원치 않은 lock 유지. 항상 commit / rollback.

### 함정 6 — 인덱스 없는 UPDATE/DELETE
모든 행 락. 또는 처음부터 PK 로 범위 좁히기.

### 함정 7 — `LOCK IN SHARE MODE` 오용
공유 락이 다른 트랜잭션의 UPDATE 와 함께 deadlock 의 주범.

### 함정 8 — `KILL` 의 무익
ROLLBACK 단계가 길면 KILL 후에도 시간 ↑.

---

## 15. 학습 자료

- **High Performance MySQL** Ch. 1, 6
- **MySQL Reference Manual** Ch. 17.7 (InnoDB Locking)
- **Percona Blog** — Gap Lock 관련

---

## 16. 관련

- [[innodb-engine]] — 락 구조
- [[explain]] — 락 진단
- [[performance-tuning]] — 트랜잭션 튜닝
- [[mysql]] — MySQL hub
