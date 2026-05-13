---
title: "Cassandra CQL 문법 — Keyspace / Table / DML / LWT"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:20:00+09:00
tags:
  - database
  - cassandra
  - cql
---

# Cassandra CQL 문법

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DDL / DML / LWT / BATCH |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. Keyspace

```sql
CREATE KEYSPACE app
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3, 'dc2': 3
}
AND DURABLE_WRITES = true;

USE app;

ALTER KEYSPACE app WITH REPLICATION = {...};
DROP KEYSPACE app;

DESC KEYSPACES;
DESC KEYSPACE app;
```

---

## 2. 데이터 타입

| 타입 | 의미 |
| --- | --- |
| `TEXT`, `VARCHAR` | UTF-8 |
| `ASCII` | ASCII |
| `INT`, `BIGINT`, `SMALLINT`, `TINYINT` | 정수 |
| `FLOAT`, `DOUBLE`, `DECIMAL`, `VARINT` | 부동 / 임의 |
| `BOOLEAN` | T/F |
| `BLOB` | 바이트 |
| `UUID` | UUID v4 |
| `TIMEUUID` | UUID v1 (시간) |
| `DATE`, `TIME`, `TIMESTAMP` | 날짜 / 시간 |
| `DURATION` | 기간 (3.10+) |
| `INET` | IP |
| `COUNTER` | 카운터 (특별) |
| `LIST<T>`, `SET<T>`, `MAP<K,V>` | 컬렉션 |
| `FROZEN<X>` | 변경 불가 (통째로) |
| `TUPLE<T1,T2,...>` | 튜플 |
| `UDT` | 사용자 정의 |

---

## 3. 테이블

```sql
CREATE TABLE users (
  user_id    UUID PRIMARY KEY,
  email      TEXT,
  name       TEXT,
  created_at TIMESTAMP
);

-- Composite
CREATE TABLE events (
  user_id     UUID,
  event_time  TIMESTAMP,
  event_id    UUID,
  type        TEXT,
  data        TEXT,
  PRIMARY KEY ((user_id), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id ASC)
  AND default_time_to_live = 7776000      -- 90 days
  AND gc_grace_seconds = 864000
  AND compaction = {'class':'TimeWindowCompactionStrategy',
                    'compaction_window_size':1,
                    'compaction_window_unit':'DAYS'};
```

### 3.1 ALTER

```sql
ALTER TABLE users ADD nickname TEXT;
ALTER TABLE users DROP nickname;
ALTER TABLE users RENAME old TO new;
ALTER TABLE users WITH default_time_to_live = 86400;
ALTER TABLE users WITH gc_grace_seconds = 0;
```

타입 변경 X. PK 변경 X.

### 3.2 DROP

```sql
DROP TABLE users;
TRUNCATE users;
```

---

## 4. INSERT / UPDATE / DELETE

```sql
INSERT INTO users (user_id, email, name)
VALUES (uuid(), 'a@x.com', 'A');

INSERT INTO users (...) VALUES (...)
USING TTL 86400;                          -- 24h
USING TIMESTAMP 1700000000000000;         -- μs (조건적 쓰기 시)
IF NOT EXISTS;                            -- LWT — 비쌈

UPDATE users SET name = 'B', email = 'b@x.com'
WHERE user_id = 550e...;

UPDATE users USING TTL 3600 SET ... WHERE ...;

UPDATE users SET name = 'X' WHERE user_id = ?
IF name = 'old';                          -- LWT

DELETE FROM users WHERE user_id = 550e...;
DELETE name FROM users WHERE user_id = ?;   -- 컬럼만 NULL
DELETE FROM events WHERE user_id = ? AND event_time = ?;
```

⚠️ INSERT 와 UPDATE 는 거의 동일 — 둘 다 upsert.

---

## 5. SELECT

```sql
SELECT * FROM users WHERE user_id = ?;
SELECT id, email FROM users WHERE user_id = ?;

-- partition + range
SELECT * FROM events
WHERE user_id = ?
  AND event_time > '2026-05-01'
  AND event_time < '2026-06-01'
ORDER BY event_time DESC
LIMIT 100;

-- IN
SELECT * FROM users WHERE user_id IN (uuid1, uuid2, uuid3);

-- TOKEN — 거의 안 씀 (전체 scan)
SELECT * FROM users WHERE TOKEN(user_id) > TOKEN(?);

-- ALLOW FILTERING ⚠️
SELECT * FROM events WHERE type = 'login' ALLOW FILTERING;
```

### 5.1 PAGING

cqlsh:
```
PAGING 100;
```

드라이버는 자동 페이징.

---

## 6. Collection 조작

### 6.1 SET

```sql
UPDATE users SET tags = tags + {'admin'} WHERE user_id = ?;
UPDATE users SET tags = tags - {'old'} WHERE user_id = ?;
UPDATE users SET tags = {'a','b'} WHERE user_id = ?;     -- 통째로
```

### 6.2 LIST

```sql
UPDATE users SET roles = roles + ['admin'] WHERE user_id = ?;
UPDATE users SET roles = roles - ['old'] WHERE user_id = ?;
UPDATE users SET roles[0] = 'new' WHERE user_id = ?;
DELETE roles[0] FROM users WHERE user_id = ?;
```

### 6.3 MAP

```sql
UPDATE users SET attrs = attrs + {'theme':'dark'} WHERE user_id = ?;
UPDATE users SET attrs['theme'] = 'light' WHERE user_id = ?;
DELETE attrs['theme'] FROM users WHERE user_id = ?;
```

---

## 7. UDT (User-Defined Type)

```sql
CREATE TYPE address (street TEXT, city TEXT, zip TEXT);

CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  home    FROZEN<address>,
  addrs   LIST<FROZEN<address>>
);

INSERT INTO users (user_id, home)
VALUES (uuid(), {street:'1 Main', city:'Seoul', zip:'12345'});

UPDATE users SET home.city = 'Busan' WHERE user_id = ?;   -- 일부만
```

---

## 8. Counter

```sql
CREATE TABLE page_counter (
  page_id UUID PRIMARY KEY,
  views   COUNTER,
  likes   COUNTER
);

UPDATE page_counter SET views = views + 1 WHERE page_id = ?;
UPDATE page_counter SET likes = likes - 1 WHERE page_id = ?;

-- INSERT X — UPDATE 만
-- 다른 컬럼 / 일반 UPDATE 와 섞기 X
```

---

## 9. BATCH

```sql
BEGIN BATCH
  INSERT INTO messages_by_user (...) VALUES (...);
  INSERT INTO messages_by_chat (...) VALUES (...);
APPLY BATCH;
```

| 종류 | 의미 |
| --- | --- |
| `BEGIN BATCH` (Logged) | 원자성 (실패 시 모두 적용 후 retry) |
| `BEGIN UNLOGGED BATCH` | 성능, 원자성 X |
| `BEGIN COUNTER BATCH` | Counter 만 |

⚠️ multi-partition batch 는 비쌈. **같은 partition** 만 권장.

---

## 10. LWT (Lightweight Transaction)

Paxos 기반 — 4 round trip.

```sql
-- 유일성
INSERT INTO users (...) VALUES (...) IF NOT EXISTS;

-- 조건부
UPDATE users SET email = ? WHERE user_id = ?
IF email = ?;

-- DELETE
DELETE FROM users WHERE user_id = ?
IF EXISTS;

-- 다중 조건
UPDATE users SET role = ?
WHERE user_id = ?
IF role = 'guest' AND active = true;
```

→ 결과: `[applied] = true/false`.

---

## 11. Index / View

```sql
CREATE INDEX idx_users_email ON users (email);            -- 신중
CREATE CUSTOM INDEX ... USING 'sai';                       -- SAI (5.0+)

DROP INDEX idx_users_email;

-- Materialized View (시험적)
CREATE MATERIALIZED VIEW users_by_email AS
  SELECT * FROM users
  WHERE email IS NOT NULL AND user_id IS NOT NULL
  PRIMARY KEY (email, user_id);
```

자세히 → [[data-modeling]]

---

## 12. UDF / UDA (User-Defined Function / Aggregate)

```sql
CREATE FUNCTION sum_two (a int, b int)
RETURNS NULL ON NULL INPUT
RETURNS int
LANGUAGE java
AS 'return a + b;';

SELECT sum_two(a, b) FROM ...;
```

기본 비활성 (`enable_user_defined_functions: true`). Java / Scala.

---

## 13. 시간 함수

```sql
SELECT now();                       -- TIMEUUID
SELECT toUnixTimestamp(now());
SELECT toTimestamp(now());
SELECT dateOf(now());
SELECT minTimeuuid('2026-05-13 00:00:00');
SELECT maxTimeuuid('2026-05-13 00:00:00');

-- 쿼리에서
WHERE id > minTimeuuid('2026-05-01');
```

---

## 14. JSON

```sql
INSERT INTO users JSON '{"user_id":"550e...","email":"a@x.com"}';

SELECT JSON * FROM users WHERE user_id = ?;
```

JSON 으로 입출력 — 클라이언트 편의.

---

## 15. 권한

```sql
CREATE ROLE alice WITH PASSWORD = 'secret' AND LOGIN = true;
ALTER ROLE alice WITH PASSWORD = 'new';
DROP ROLE alice;

GRANT SELECT ON KEYSPACE app TO alice;
GRANT MODIFY ON app.users TO alice;
GRANT EXECUTE ON FUNCTION app.f TO alice;
REVOKE MODIFY ON app.users FROM alice;

LIST ALL PERMISSIONS OF alice;
LIST ROLES;
```

---

## 16. 함정

### 16.1 ALLOW FILTERING 운영
거의 항상 잘못된 데이터 모델 신호.

### 16.2 LWT 남용
4 round trip. 정말 필요한 곳만.

### 16.3 multi-partition BATCH
같은 partition 만.

### 16.4 Counter / 일반 컬럼 섞기
Counter 테이블엔 Counter 만.

### 16.5 TTL on Counter
Counter 에 TTL X.

### 16.6 Collection 큰 사이즈
수십~수백 element 까지. 큰 데이터 별도 테이블.

### 16.7 IN 큰 리스트
1 노드만 hit 하려면 partition key. 여러 partition IN → scatter.

### 16.8 ORDER BY 다른 clustering
정의된 순서만 가능 (또는 reverse).

---

## 17. 학습 자료

- **CQL Reference** — cassandra.apache.org/doc/latest/cassandra/cql
- **DataStax Academy** — CQL course
- **Cassandra Cookbook**

---

## 18. 관련

- [[data-modeling]] — partition / clustering
- [[consistency-tunable]] — CL
- [[cassandra]] — Cassandra hub
