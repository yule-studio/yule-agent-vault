---
title: "PostgreSQL SQL 문법 — DDL / DML / 윈도우 / CTE"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:20:00+09:00
tags:
  - database
  - postgresql
  - sql
---

# PostgreSQL SQL 문법

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DDL / DML / CTE / 윈도우 / MERGE |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. SQL 분류

| 분류 | 의미 | 예 |
| --- | --- | --- |
| **DDL** | Data Definition | CREATE, ALTER, DROP, TRUNCATE |
| **DML** | Data Manipulation | INSERT, UPDATE, DELETE, MERGE |
| **DQL** | Data Query | SELECT |
| **DCL** | Data Control | GRANT, REVOKE |
| **TCL** | Transaction Control | BEGIN, COMMIT, ROLLBACK, SAVEPOINT |

---

## 2. DDL — 테이블 / 스키마

### 2.1 CREATE TABLE

```sql
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    name        TEXT,
    age         INTEGER CHECK (age >= 0),
    role        TEXT NOT NULL DEFAULT 'user',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ
);
```

### 2.2 외래키

```sql
CREATE TABLE posts (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id   BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title     TEXT NOT NULL
);
```

| 옵션 | 동작 |
| --- | --- |
| `CASCADE` | 부모 삭제 시 자식도 삭제 |
| `SET NULL` | 자식 FK 를 NULL 로 |
| `SET DEFAULT` | 자식 FK 를 default 로 |
| `RESTRICT` (기본) | 부모 삭제 막음 (즉시) |
| `NO ACTION` | RESTRICT 와 비슷 (트랜잭션 끝에서 검사) |

### 2.3 ALTER TABLE

```sql
ALTER TABLE users ADD COLUMN nickname TEXT;
ALTER TABLE users DROP COLUMN nickname;
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;
ALTER TABLE users ALTER COLUMN role SET DEFAULT 'guest';
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;
ALTER TABLE users ADD CONSTRAINT chk_age CHECK (age < 150);
ALTER TABLE users DROP CONSTRAINT chk_age;
```

### 2.4 스키마

```sql
CREATE SCHEMA analytics;
SET search_path = analytics, public;
CREATE TABLE analytics.events (...);
```

---

## 3. DML — INSERT / UPDATE / DELETE

### 3.1 INSERT

```sql
INSERT INTO users (email, name) VALUES
  ('a@x.com', 'A'),
  ('b@x.com', 'B');

-- RETURNING — 생성된 값 받기
INSERT INTO users (email, name) VALUES ('c@x.com', 'C')
RETURNING id, created_at;

-- UPSERT (ON CONFLICT)
INSERT INTO users (email, name) VALUES ('a@x.com', 'A2')
ON CONFLICT (email) DO UPDATE
SET name = EXCLUDED.name,
    updated_at = now();

-- 충돌 시 무시
INSERT INTO users (email, name) VALUES ('a@x.com', 'A2')
ON CONFLICT DO NOTHING;
```

### 3.2 UPDATE

```sql
UPDATE users
SET role = 'admin', updated_at = now()
WHERE id = 1
RETURNING id, role;

-- JOIN 으로 업데이트
UPDATE posts p
SET author_name = u.name
FROM users u
WHERE p.user_id = u.id;
```

### 3.3 DELETE

```sql
DELETE FROM users WHERE id = 1 RETURNING *;

DELETE FROM posts p
USING users u
WHERE p.user_id = u.id AND u.role = 'banned';

-- 전체 삭제 — TRUNCATE 가 빠름
TRUNCATE TABLE logs;
TRUNCATE TABLE logs RESTART IDENTITY CASCADE;
```

### 3.4 MERGE (PG 15+)

```sql
MERGE INTO inventory AS t
USING new_arrivals AS s
ON t.product_id = s.product_id
WHEN MATCHED THEN
  UPDATE SET qty = t.qty + s.qty
WHEN NOT MATCHED THEN
  INSERT (product_id, qty) VALUES (s.product_id, s.qty);
```

---

## 4. SELECT — 기초

```sql
SELECT id, email FROM users;
SELECT * FROM users WHERE age >= 18 AND role = 'admin';
SELECT * FROM users ORDER BY created_at DESC LIMIT 10 OFFSET 20;
SELECT DISTINCT role FROM users;
SELECT COUNT(*) FROM users;
```

### 4.1 패턴 매칭

```sql
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT * FROM users WHERE email ILIKE 'A%';    -- 대소문자 무시
SELECT * FROM users WHERE email ~* '^a.*\.com$';  -- 정규식
SELECT * FROM users WHERE email SIMILAR TO 'a%(b|c)';
```

### 4.2 NULL

```sql
SELECT * FROM users WHERE name IS NULL;
SELECT * FROM users WHERE name IS NOT NULL;
SELECT COALESCE(name, email, 'unknown') FROM users;
SELECT NULLIF(name, '') FROM users;            -- '' → NULL
```

---

## 5. JOIN

```sql
-- INNER
SELECT u.name, p.title
FROM users u
JOIN posts p ON p.user_id = u.id;

-- LEFT (NULL 포함)
SELECT u.name, p.title
FROM users u
LEFT JOIN posts p ON p.user_id = u.id;

-- FULL OUTER
SELECT u.name, p.title
FROM users u
FULL OUTER JOIN posts p ON p.user_id = u.id;

-- CROSS (카르테시안)
SELECT * FROM colors CROSS JOIN sizes;

-- LATERAL — 좌측 행마다 우측 평가
SELECT u.id, recent.title
FROM users u,
LATERAL (
  SELECT title FROM posts p
  WHERE p.user_id = u.id
  ORDER BY created_at DESC LIMIT 3
) recent;
```

---

## 6. GROUP BY / HAVING / 집계

```sql
SELECT role, COUNT(*) AS cnt, AVG(age) AS avg_age
FROM users
GROUP BY role
HAVING COUNT(*) > 10
ORDER BY cnt DESC;
```

### 6.1 집계 함수

```sql
COUNT(*) / COUNT(col)
SUM, AVG, MIN, MAX
STDDEV, VARIANCE
STRING_AGG(name, ',' ORDER BY name)
ARRAY_AGG(name)
JSON_AGG(row_to_json(u.*))
JSONB_OBJECT_AGG(key, value)
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)   -- 중위값
```

### 6.2 FILTER

```sql
SELECT
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE role = 'admin') AS admins,
  AVG(age) FILTER (WHERE active) AS avg_active_age
FROM users;
```

### 6.3 ROLLUP / CUBE / GROUPING SETS

```sql
SELECT region, product, SUM(sales)
FROM orders
GROUP BY ROLLUP (region, product);

-- region, product 별 + region 소계 + 총계
```

---

## 7. 윈도우 함수

```sql
SELECT
  id, user_id, amount, created_at,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS rn,
  RANK()       OVER (PARTITION BY user_id ORDER BY amount DESC) AS rk,
  LAG(amount)  OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amt,
  SUM(amount)  OVER (PARTITION BY user_id ORDER BY created_at
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_sum
FROM orders;
```

### 7.1 주요 윈도우 함수

| 함수 | 의미 |
| --- | --- |
| `ROW_NUMBER()` | 순번 (1,2,3) |
| `RANK()` | 순위 (동점 후 건너뜀: 1,2,2,4) |
| `DENSE_RANK()` | 순위 (건너뛰지 않음: 1,2,2,3) |
| `LAG(col, n)` | n 행 이전 값 |
| `LEAD(col, n)` | n 행 이후 |
| `FIRST_VALUE` / `LAST_VALUE` | 윈도우 첫/마지막 |
| `NTILE(n)` | n 등분 (사분위 등) |
| `PERCENT_RANK` | 누적 백분율 |

### 7.2 가장 흔한 패턴 — Top-N per group

```sql
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM posts
) t
WHERE rn <= 3;
```

---

## 8. CTE (Common Table Expression)

```sql
WITH recent_users AS (
  SELECT * FROM users WHERE created_at > now() - INTERVAL '7 days'
),
recent_posts AS (
  SELECT * FROM posts WHERE created_at > now() - INTERVAL '7 days'
)
SELECT u.email, COUNT(p.id) AS posts
FROM recent_users u
LEFT JOIN recent_posts p ON p.user_id = u.id
GROUP BY u.email;
```

### 8.1 Recursive CTE — 트리

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 0 AS depth
  FROM categories
  WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c
  JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY depth, id;
```

### 8.2 Writable CTE

```sql
WITH moved AS (
  DELETE FROM users WHERE created_at < now() - INTERVAL '1 year'
  RETURNING *
)
INSERT INTO users_archive SELECT * FROM moved;
```

---

## 9. 서브쿼리

```sql
-- 스칼라
SELECT name, (SELECT COUNT(*) FROM posts p WHERE p.user_id = u.id) AS posts
FROM users u;

-- EXISTS — 가장 효율적 (대부분)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM posts p WHERE p.user_id = u.id);

-- IN
SELECT * FROM users WHERE id IN (SELECT user_id FROM posts);

-- NOT EXISTS — NULL 안전 (NOT IN 함정 회피)
```

⚠️ `WHERE col NOT IN (SELECT ... )` 는 서브쿼리에 NULL 이 하나라도 있으면 빈 결과. `NOT EXISTS` 권장.

---

## 10. SET 연산

```sql
SELECT email FROM users
UNION
SELECT email FROM newsletter_signups;

-- UNION ALL (중복 제거 X, 빠름)
-- INTERSECT — 교집합
-- EXCEPT — 차집합
```

---

## 11. CASE / CONDITIONAL

```sql
SELECT
  name,
  CASE
    WHEN age < 18 THEN 'minor'
    WHEN age < 65 THEN 'adult'
    ELSE 'senior'
  END AS group
FROM users;

-- 짧은 형식
SELECT CASE role WHEN 'admin' THEN 'A' ELSE 'U' END FROM users;
```

---

## 12. 함수 / 프로시저

```sql
CREATE OR REPLACE FUNCTION user_full_name(u users)
RETURNS TEXT
LANGUAGE SQL IMMUTABLE
AS $$ SELECT u.name || ' (' || u.email || ')' $$;

SELECT user_full_name(u.*) FROM users u;
```

### 12.1 PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION increment_login(p_user_id BIGINT)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
  v_count INTEGER;
BEGIN
  UPDATE users SET login_count = login_count + 1
  WHERE id = p_user_id
  RETURNING login_count INTO v_count;
  RETURN v_count;
END
$$;
```

---

## 13. 트리거

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END $$;

CREATE TRIGGER users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 14. 트랜잭션

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- SAVEPOINT
BEGIN;
  INSERT INTO orders ...;
  SAVEPOINT before_items;
  INSERT INTO items ...;
  ROLLBACK TO before_items;   -- items 만 취소
COMMIT;
```

자세히 → [[transactions-mvcc]]

---

## 15. 권한

```sql
GRANT SELECT ON users TO app_ro;
GRANT INSERT, UPDATE ON users TO app_rw;
GRANT USAGE ON SCHEMA public TO app_ro;

REVOKE INSERT ON users FROM app_ro;

-- Row Level Security (RLS)
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_owns_row ON users
  USING (id = current_setting('app.user_id')::BIGINT);
```

---

## 16. 함정

### 함정 1 — `NOT IN` + NULL
서브쿼리에 NULL 하나라도 → 결과 비어짐. `NOT EXISTS` 사용.

### 함정 2 — `SELECT *` 운영
컬럼 추가 시 깨짐. 명시적 컬럼 권장.

### 함정 3 — `OFFSET` 페이지네이션
큰 OFFSET 은 느림 — keyset (`WHERE id > last_id LIMIT n`) 권장.

### 함정 4 — `=` vs `IS NULL`
`col = NULL` 은 항상 false. `IS NULL` 사용.

### 함정 5 — `LIMIT` 없는 ORDER BY
필요한 경우 외에는 LIMIT 같이.

### 함정 6 — Implicit cast
`WHERE int_col = '123'` 동작은 하지만 인덱스 안 탈 수 있음.

---

## 17. 관련

- [[indexes]] — 인덱스
- [[explain-analyze]] — 실행 계획
- [[transactions-mvcc]] — 트랜잭션
- [[postgresql]] — PostgreSQL hub
