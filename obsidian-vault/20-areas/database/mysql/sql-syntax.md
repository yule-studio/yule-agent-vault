---
title: "MySQL SQL 문법 — DDL / DML / Window / CTE"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:30:00+09:00
tags:
  - database
  - mysql
  - sql
---

# MySQL SQL 문법

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DDL / DML / CTE / Window |

**[[mysql|↑ MySQL hub]]**

---

## 1. DDL

### 1.1 CREATE TABLE

```sql
CREATE TABLE users (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    email       VARCHAR(320) NOT NULL,
    name        VARCHAR(100),
    age         INT UNSIGNED CHECK (age < 150),
    role        VARCHAR(20) NOT NULL DEFAULT 'user',
    created_at  DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at  DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                  ON UPDATE CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    UNIQUE KEY uk_users_email (email),
    KEY idx_users_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

⚠️ MySQL 의 `CHECK` 는 8.0.16+ 부터 실제 작동.

### 1.2 외래키

```sql
CREATE TABLE posts (
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id  BIGINT UNSIGNED NOT NULL,
    title    VARCHAR(200) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_posts_user (user_id),
    CONSTRAINT fk_posts_user
      FOREIGN KEY (user_id) REFERENCES users(id)
      ON DELETE CASCADE
      ON UPDATE CASCADE
) ENGINE=InnoDB;
```

### 1.3 ALTER TABLE

```sql
ALTER TABLE users ADD COLUMN nickname VARCHAR(100);
ALTER TABLE users DROP COLUMN nickname;
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users MODIFY COLUMN age BIGINT UNSIGNED;
ALTER TABLE users ALTER COLUMN role SET DEFAULT 'guest';
ALTER TABLE users ADD INDEX idx_users_role (role);
ALTER TABLE users DROP INDEX idx_users_role;

-- 여러 변경 한 번에 (online DDL 최적)
ALTER TABLE users
  ADD COLUMN birthday DATE,
  ADD INDEX idx_birthday (birthday),
  ALGORITHM=INPLACE, LOCK=NONE;
```

### 1.4 Online DDL — ALGORITHM / LOCK

```sql
ALTER TABLE users ADD COLUMN x INT, ALGORITHM=INPLACE, LOCK=NONE;
```

| ALGORITHM | 의미 |
| --- | --- |
| `INSTANT` | 즉시 (메타데이터만) — 8.0+, 컬럼 추가 등 |
| `INPLACE` | 테이블 복사 X, 일부 lock |
| `COPY` | 전체 복사 (느림, lock) |

| LOCK | 의미 |
| --- | --- |
| `NONE` | DML 동시 가능 |
| `SHARED` | SELECT 만 |
| `EXCLUSIVE` | 모두 막음 |

→ 가능하면 `INSTANT` / `INPLACE + NONE`. 안 되면 **pt-online-schema-change / gh-ost**.

---

## 2. DML

### 2.1 INSERT

```sql
INSERT INTO users (email, name) VALUES
  ('a@x.com', 'A'),
  ('b@x.com', 'B');

-- ON DUPLICATE KEY UPDATE
INSERT INTO users (email, name) VALUES ('a@x.com', 'A2')
ON DUPLICATE KEY UPDATE name = VALUES(name);
-- 8.0+: VALUES() 대신 alias
INSERT INTO users (email, name) VALUES ('a@x.com', 'A2') AS new
ON DUPLICATE KEY UPDATE name = new.name;

-- 무시
INSERT IGNORE INTO users (email, name) VALUES ('a@x.com', 'A2');

-- 또는 REPLACE (DELETE + INSERT — 위험)
REPLACE INTO users (email, name) VALUES ('a@x.com', 'A2');
```

### 2.2 UPDATE

```sql
UPDATE users SET role = 'admin', updated_at = NOW(6) WHERE id = 1;

-- JOIN UPDATE
UPDATE posts p
JOIN users u ON p.user_id = u.id
SET p.author_name = u.name;

-- LIMIT 안전망
UPDATE users SET role = 'guest' WHERE age < 18 LIMIT 1000;
```

### 2.3 DELETE

```sql
DELETE FROM users WHERE id = 1;

-- JOIN DELETE
DELETE p FROM posts p
JOIN users u ON p.user_id = u.id
WHERE u.role = 'banned';

-- TRUNCATE
TRUNCATE TABLE logs;
```

---

## 3. SELECT 기초

```sql
SELECT id, email FROM users WHERE age >= 18 ORDER BY created_at DESC LIMIT 10 OFFSET 20;
SELECT DISTINCT role FROM users;
SELECT COUNT(*), AVG(age) FROM users;

-- 패턴
WHERE email LIKE '%@example.com'
WHERE email REGEXP '^[a-z]+@'

-- NULL
WHERE name IS NULL
SELECT COALESCE(name, email, 'unknown')
SELECT NULLIF(name, '')
```

---

## 4. JOIN

```sql
SELECT u.name, p.title
FROM users u
JOIN posts p ON p.user_id = u.id;

LEFT JOIN ...
RIGHT JOIN ...
-- FULL OUTER JOIN 없음 → UNION 으로 구현
SELECT u.id, p.id FROM users u LEFT JOIN posts p ON p.user_id = u.id
UNION
SELECT u.id, p.id FROM users u RIGHT JOIN posts p ON p.user_id = u.id;

CROSS JOIN

-- STRAIGHT_JOIN — 옵티마이저 순서 강제
SELECT STRAIGHT_JOIN ... FROM small s JOIN big b ON ...
```

---

## 5. GROUP BY / HAVING

```sql
SELECT role, COUNT(*) AS cnt, AVG(age) AS avg_age
FROM users
GROUP BY role
HAVING cnt > 10
ORDER BY cnt DESC;
```

### 5.1 ONLY_FULL_GROUP_BY

```sql
-- ❌ 에러
SELECT role, name FROM users GROUP BY role;

-- ✅ 집계 또는 functional dependency
SELECT role, ANY_VALUE(name) FROM users GROUP BY role;
SELECT role, MIN(name) FROM users GROUP BY role;
```

### 5.2 ROLLUP

```sql
SELECT region, product, SUM(sales)
FROM orders
GROUP BY region, product WITH ROLLUP;
```

---

## 6. Window 함수 (8.0+)

```sql
SELECT
  id, user_id, amount, created_at,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS rn,
  RANK()       OVER (PARTITION BY user_id ORDER BY amount DESC) AS rk,
  LAG(amount)  OVER (PARTITION BY user_id ORDER BY created_at) AS prev,
  SUM(amount)  OVER (PARTITION BY user_id ORDER BY created_at
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling
FROM orders;
```

| 함수 | 의미 |
| --- | --- |
| `ROW_NUMBER` / `RANK` / `DENSE_RANK` | 순번 / 순위 |
| `LAG` / `LEAD` | 이전 / 이후 행 |
| `FIRST_VALUE` / `LAST_VALUE` | 윈도우 첫 / 마지막 |
| `NTILE(n)` | n 등분 |

### 6.1 Top-N per group

```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM posts
) t WHERE rn <= 3;
```

---

## 7. CTE (8.0+)

```sql
WITH recent AS (
  SELECT * FROM users WHERE created_at > NOW() - INTERVAL 7 DAY
)
SELECT * FROM recent WHERE role = 'admin';
```

### 7.1 Recursive

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 0 AS depth
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

---

## 8. 서브쿼리

```sql
-- 스칼라
SELECT name, (SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id) AS posts
FROM users;

-- EXISTS (효율적)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM posts WHERE posts.user_id = u.id);

-- IN (소량 OK)
SELECT * FROM users WHERE id IN (SELECT user_id FROM posts);

-- NOT IN + NULL 함정 → NOT EXISTS 권장
```

---

## 9. SET 연산

```sql
SELECT email FROM users
UNION
SELECT email FROM newsletter;

UNION ALL    -- 중복 OK
INTERSECT    -- 8.0.31+
EXCEPT       -- 8.0.31+
```

---

## 10. CASE

```sql
SELECT name,
  CASE
    WHEN age < 18 THEN 'minor'
    WHEN age < 65 THEN 'adult'
    ELSE 'senior'
  END AS grp
FROM users;

SELECT IF(active, 'Y', 'N') FROM users;
SELECT IFNULL(name, '?') FROM users;
```

---

## 11. 트랜잭션 / 락

```sql
START TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 또는 BEGIN;
-- ROLLBACK;

-- SAVEPOINT
SAVEPOINT s1;
ROLLBACK TO s1;
RELEASE SAVEPOINT s1;

-- 행 락
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;   -- 큐
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;       -- 옛
SELECT * FROM accounts WHERE id = 1 FOR SHARE;                -- 8.0+
```

자세히 → [[transactions]]

---

## 12. 권한

```sql
GRANT SELECT, INSERT ON app.* TO 'app_rw'@'%';
REVOKE INSERT ON app.* FROM 'app_rw'@'%';
SHOW GRANTS FOR 'app_rw'@'%';

-- Role (8.0+)
CREATE ROLE 'app_reader';
GRANT SELECT ON app.* TO 'app_reader';
GRANT 'app_reader' TO 'alice'@'%';
SET DEFAULT ROLE 'app_reader' TO 'alice'@'%';
```

---

## 13. 저장 프로시저 / 함수

```sql
DELIMITER //
CREATE PROCEDURE increment_login(IN p_user_id BIGINT)
BEGIN
  UPDATE users SET login_count = login_count + 1 WHERE id = p_user_id;
END //
DELIMITER ;

CALL increment_login(1);
```

```sql
CREATE FUNCTION full_name(first VARCHAR(50), last VARCHAR(50))
RETURNS VARCHAR(120) DETERMINISTIC
RETURN CONCAT(first, ' ', last);
```

---

## 14. 트리거

```sql
CREATE TRIGGER trg_users_updated
BEFORE UPDATE ON users
FOR EACH ROW
SET NEW.updated_at = NOW(6);
```

---

## 15. View

```sql
CREATE VIEW active_users AS
SELECT * FROM users WHERE active = 1;

CREATE OR REPLACE VIEW v_user_summary AS
SELECT u.id, u.name, COUNT(p.id) AS posts
FROM users u LEFT JOIN posts p ON p.user_id = u.id
GROUP BY u.id;
```

---

## 16. 함정

### 함정 1 — `NOT IN` + NULL
빈 결과. `NOT EXISTS`.

### 함정 2 — DDL 은 자동 COMMIT
`ALTER TABLE` 가 트랜잭션 안에서도 자동 commit. 롤백 불가.

### 함정 3 — `SELECT *` 의 컬럼 순서 의존

### 함정 4 — `ONLY_FULL_GROUP_BY` 위반
SELECT 컬럼은 GROUP BY 또는 집계여야.

### 함정 5 — `REPLACE INTO`
DELETE + INSERT — 새 PK, AUTO_INCREMENT 폭증, FK CASCADE 폭주. `ON DUPLICATE KEY UPDATE` 권장.

### 함정 6 — `LIMIT` 의 OFFSET
큰 OFFSET 은 느림. keyset 페이지네이션.

### 함정 7 — Implicit Type Conversion
`WHERE int_col = '123'` 인덱스 안 탈 수도.

---

## 17. 관련

- [[indexes]] — 인덱스
- [[explain]] — 실행 계획
- [[transactions]] — 트랜잭션
- [[mysql]] — MySQL hub
