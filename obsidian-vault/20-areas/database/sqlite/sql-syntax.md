---
title: "SQLite SQL 문법 — DDL / DML / Window / CTE / FTS / JSON"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:35:00+09:00
tags:
  - database
  - sqlite
  - sql
---

# SQLite SQL 문법

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DDL / DML / Window / CTE / FTS5 |

**[[sqlite|↑ SQLite hub]]**

---

## 1. DDL

### 1.1 CREATE TABLE

```sql
CREATE TABLE users (
  id          INTEGER PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,
  name        TEXT,
  age         INTEGER CHECK (age >= 0),
  role        TEXT NOT NULL DEFAULT 'user',
  data        BLOB,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
) STRICT;
```

### 1.2 외래키

```sql
CREATE TABLE posts (
  id      INTEGER PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title   TEXT NOT NULL
);

-- ⚠️ FK 활성 필수
PRAGMA foreign_keys = ON;
```

### 1.3 ALTER TABLE

SQLite 의 ALTER 는 제한적:

```sql
ALTER TABLE users ADD COLUMN nickname TEXT;
ALTER TABLE users DROP COLUMN nickname;          -- 3.35+
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users RENAME TO users_v2;
```

❌ 컬럼 타입 변경 / 제약 변경 → **재생성 패턴**:

```sql
BEGIN;
CREATE TABLE users_new (...);
INSERT INTO users_new SELECT ... FROM users;
DROP TABLE users;
ALTER TABLE users_new RENAME TO users;
COMMIT;
```

`legacy_alter_table = OFF` (기본) 면 view / trigger / FK 가 자동 갱신.

### 1.4 인덱스

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_email_u ON users(email);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 부분 인덱스
CREATE INDEX idx_users_active ON users(email) WHERE active = 1;

-- Expression
CREATE INDEX idx_users_lower ON users(LOWER(email));

DROP INDEX idx_users_email;
```

### 1.5 VIEW

```sql
CREATE VIEW active_users AS
  SELECT * FROM users WHERE active = 1;

DROP VIEW active_users;
```

### 1.6 TRIGGER

```sql
CREATE TRIGGER trg_users_updated
AFTER UPDATE ON users
BEGIN
  UPDATE users SET updated_at = datetime('now') WHERE id = NEW.id;
END;
```

---

## 2. DML

### 2.1 INSERT

```sql
INSERT INTO users (email, name) VALUES
  ('a@x.com', 'A'),
  ('b@x.com', 'B');

-- RETURNING (3.35+)
INSERT INTO users (email, name) VALUES ('c@x.com', 'C')
RETURNING id, created_at;

-- UPSERT (3.24+)
INSERT INTO users (email, name) VALUES ('a@x.com', 'A2')
ON CONFLICT(email) DO UPDATE SET name = excluded.name;

-- OR IGNORE / OR REPLACE
INSERT OR IGNORE INTO users (email, name) VALUES ('a@x.com', 'A');
INSERT OR REPLACE INTO users (id, email) VALUES (1, 'new@x.com');
```

### 2.2 UPDATE

```sql
UPDATE users SET role = 'admin' WHERE id = 1;

-- FROM JOIN (3.33+)
UPDATE posts SET author = u.name
FROM users u
WHERE posts.user_id = u.id;

UPDATE OR IGNORE users SET email = 'dup@x.com' WHERE id = 2;
```

### 2.3 DELETE

```sql
DELETE FROM users WHERE id = 1 RETURNING *;

-- LIMIT (3.6+ compile option, 보통 활성)
DELETE FROM logs WHERE created_at < '2026-01-01' LIMIT 10000;
```

---

## 3. SELECT

```sql
SELECT id, email FROM users WHERE age >= 18 ORDER BY created_at DESC LIMIT 10;
SELECT DISTINCT role FROM users;
SELECT COUNT(*), AVG(age) FROM users;
```

### 3.1 패턴

```sql
WHERE email LIKE '%@example.com'
WHERE email GLOB 'a*'                  -- 와일드카드 다른 문법
WHERE email REGEXP '^a.+\.com$'         -- regexp 함수 등록 시
```

### 3.2 NULL

```sql
WHERE name IS NULL
SELECT COALESCE(name, email)
SELECT IFNULL(name, 'unknown')
SELECT NULLIF(name, '')
```

---

## 4. JOIN

```sql
-- INNER
SELECT u.name, p.title FROM users u JOIN posts p ON p.user_id = u.id;

-- LEFT
SELECT u.name, p.title FROM users u LEFT JOIN posts p ON p.user_id = u.id;

-- FULL OUTER / RIGHT (3.39+)
SELECT ... FROM a FULL OUTER JOIN b ON ...;
```

---

## 5. GROUP BY / 집계

```sql
SELECT role, COUNT(*) AS cnt, AVG(age) AS avg_age
FROM users
GROUP BY role
HAVING cnt > 10
ORDER BY cnt DESC;
```

### 5.1 집계 함수

```sql
COUNT(*), SUM, AVG, MIN, MAX
TOTAL(x)         -- SUM 의 floating 버전
GROUP_CONCAT(name, ',')
```

### 5.2 FILTER (3.30+)

```sql
SELECT
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE role = 'admin') AS admins
FROM users;
```

---

## 6. Window 함수 (3.25+)

```sql
SELECT
  id, user_id, amount, created_at,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS rn,
  RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rk,
  LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev,
  SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at
                    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling
FROM orders;
```

PG / MySQL 8.0 과 거의 동일.

---

## 7. CTE

```sql
WITH recent AS (
  SELECT * FROM users WHERE created_at > datetime('now', '-7 days')
)
SELECT * FROM recent WHERE role = 'admin';
```

### 7.1 Recursive

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 0 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

---

## 8. 트랜잭션

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- BEGIN IMMEDIATE — 즉시 write lock 획득
BEGIN IMMEDIATE;
...
COMMIT;

-- BEGIN EXCLUSIVE — 배타 락 (옛 journal mode 만 의미)
```

### 8.1 SAVEPOINT

```sql
SAVEPOINT s1;
...
ROLLBACK TO s1;
RELEASE s1;
```

자세히 → [[transactions-wal]]

---

## 9. FTS5 (Full-Text Search)

```sql
-- 가상 테이블
CREATE VIRTUAL TABLE articles_fts USING fts5(
  title, body,
  content='articles', content_rowid='id',
  tokenize='unicode61 remove_diacritics 2'
);

-- 동기화 트리거
CREATE TRIGGER articles_ai AFTER INSERT ON articles BEGIN
  INSERT INTO articles_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
CREATE TRIGGER articles_au AFTER UPDATE ON articles BEGIN
  INSERT INTO articles_fts(articles_fts, rowid, title, body)
  VALUES('delete', old.id, old.title, old.body);
  INSERT INTO articles_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
CREATE TRIGGER articles_ad AFTER DELETE ON articles BEGIN
  INSERT INTO articles_fts(articles_fts, rowid, title, body)
  VALUES('delete', old.id, old.title, old.body);
END;

-- 검색
SELECT a.* FROM articles a
JOIN articles_fts f ON f.rowid = a.id
WHERE articles_fts MATCH 'sqlite NEAR/5 fast';

-- 옵션
WHERE articles_fts MATCH 'title:sqlite OR body:fast'
ORDER BY rank
LIMIT 10;
```

⚠️ 한국어 — 기본 tokenizer 가 한글에 잘 안 됨. `unicode61` + 보조 처리 또는 외부 (signal-cli, icu).

---

## 10. JSON

```sql
SELECT json_extract(data, '$.user') FROM events;
SELECT data ->> '$.user' FROM events;       -- 3.38+ 단축

-- 함수
json_object('k','v')
json_array(1,2,3)
json_array_length(json)
json_each(json), json_tree(json)            -- 테이블 함수
json_group_array(x), json_group_object(k, v)
json_set, json_insert, json_replace, json_remove
json_valid(json)
```

### 10.1 JSONB (3.45+)

```sql
INSERT INTO events VALUES (jsonb('{"a":1}'));
SELECT jsonb_extract(data, '$.a') FROM events;
```

---

## 11. 권한 / DCL

SQLite 는 **권한 시스템 없음** — 파일 시스템의 권한이 곧 DB 권한.
다중 사용자 / 권한 분리가 필요하면 RDB 서버 사용.

---

## 12. 자주 쓰는 함수

```sql
-- 문자열
length(s), upper(s), lower(s)
substr(s, start, len)
replace(s, find, repl)
trim(s), ltrim(s), rtrim(s)
instr(s, sub)
printf('%d items', count)
hex(blob), unhex(text)
quote(value)

-- 숫자
abs, round, floor, ceil, mod
random()                    -- -2^63 ~ 2^63
randomblob(n)
zeroblob(n)

-- 날짜
datetime, date, time
strftime, julianday, unixepoch
date('now', '+1 day', 'start of month')

-- 집계
sum, avg, count, min, max, total
group_concat

-- 조건
CASE WHEN ... THEN ... ELSE ... END
iif(cond, then, else)        -- 3.32+
coalesce, ifnull, nullif
```

---

## 13. EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@x.com';
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'a@x.com';
```

EXPLAIN = VDBE bytecode (디버그).
EXPLAIN QUERY PLAN = 사람이 읽기 — `SEARCH ... USING INDEX` 등.

```
SEARCH users USING INDEX idx_users_email (email=?)     ← 좋음
SCAN users                                              ← 풀스캔
```

---

## 14. ATTACH — 여러 DB

```sql
ATTACH DATABASE 'other.db' AS other;
SELECT * FROM other.logs;
DETACH DATABASE other;

-- 트랜잭션 가능
INSERT INTO main.users SELECT * FROM other.legacy_users;
```

---

## 15. 함정

### 15.1 ALTER 제한
컬럼 변경 X. 재생성 패턴.

### 15.2 FK off
`PRAGMA foreign_keys = ON` 매 connection.

### 15.3 NULL 의 UNIQUE
여러 NULL 허용.

### 15.4 DECIMAL 미지원
정수 / TEXT.

### 15.5 `INSERT OR REPLACE` 의 부작용
DELETE 트리거 발동 + AUTOINCREMENT 변화. UPSERT 권장.

### 15.6 GLOB vs LIKE
GLOB = case-sensitive + `*?` 와일드카드. LIKE = case-insensitive + `%_`.

### 15.7 LIMIT 없는 OFFSET
의미 X. LIMIT 같이.

### 15.8 STRICT 미사용
타입 강제 X. 신규는 STRICT.

---

## 16. 학습 자료

- **SQL As Understood by SQLite** — sqlite.org/lang.html
- **FTS5** — sqlite.org/fts5.html
- **Window Functions** — sqlite.org/windowfunctions.html
- **antonz.org** — 시리즈

---

## 17. 관련

- [[data-types]]
- [[transactions-wal]]
- [[sqlite]] — SQLite hub
