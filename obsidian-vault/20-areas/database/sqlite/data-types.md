---
title: "SQLite 데이터 타입 — Affinity / STRICT"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:30:00+09:00
tags:
  - database
  - sqlite
  - data-types
---

# SQLite 데이터 타입

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Affinity / STRICT / JSON |

**[[sqlite|↑ SQLite hub]]**

---

## 1. Dynamic Typing

SQLite 는 RDB 중 거의 유일하게 **Dynamic Typing**:

```sql
CREATE TABLE t (a INTEGER);
INSERT INTO t VALUES (123);
INSERT INTO t VALUES ('hello');      -- 에러 X — text 그대로 저장!
INSERT INTO t VALUES (3.14);          -- float 그대로
```

→ 컬럼의 선언 타입은 **affinity 힌트**.

**STRICT 테이블** (3.37+) 사용 시 PG / MySQL 처럼 엄격.

---

## 2. Storage Class — 실제 저장 타입

| Class | 의미 |
| --- | --- |
| `NULL` | NULL |
| `INTEGER` | 1, 2, 3, 4, 6, 8 byte |
| `REAL` | IEEE 754 8 byte float |
| `TEXT` | UTF-8 / UTF-16 |
| `BLOB` | 바이트 그대로 |

5 가지만. 모든 값은 이 중 하나로 저장.

---

## 3. Type Affinity

선언 타입 → affinity 매핑:

| 선언 타입 (포함) | Affinity |
| --- | --- |
| `INT`, `INTEGER`, `BIGINT`, ... | INTEGER |
| `CHAR`, `VARCHAR`, `TEXT`, ... | TEXT |
| `BLOB` (또는 타입 없음) | BLOB |
| `REAL`, `FLOAT`, `DOUBLE` | REAL |
| `NUMERIC`, `DECIMAL`, `BOOL`, `DATE` | NUMERIC |

INSERT 시 affinity 에 따라 변환 시도 (실패해도 저장).

---

## 4. STRICT 테이블 (3.37+, 권장)

```sql
CREATE TABLE users (
  id    INTEGER PRIMARY KEY,
  email TEXT NOT NULL,
  age   INTEGER CHECK (age >= 0),
  data  BLOB,
  score REAL
) STRICT;
```

선언 타입 강제:
- `INTEGER`, `REAL`, `TEXT`, `BLOB`, `ANY` 만 사용 가능
- 잘못된 타입 INSERT → 에러

신규 코드는 **STRICT** 권장.

---

## 5. 자주 쓰는 타입

### 5.1 정수

```sql
id INTEGER PRIMARY KEY               -- rowid 별칭 (자동 증가)
count INTEGER NOT NULL DEFAULT 0
```

### 5.2 텍스트

```sql
email TEXT NOT NULL
name  TEXT
```

`VARCHAR(n)` 의 `(n)` 은 무시 — 길이 제한 X.

### 5.3 부동소수 / DECIMAL

SQLite 는 `DECIMAL` 타입 없음 → 정수 (100배) 또는 TEXT 로 저장.

```sql
amount_cents INTEGER NOT NULL    -- 1.5 → 150
```

### 5.4 BOOL

`BOOLEAN` 키워드 있지만 INTEGER 로 저장:

```sql
is_active INTEGER NOT NULL DEFAULT 1    -- 1/0

-- 또는 NUMERIC affinity
is_active BOOLEAN
```

### 5.5 날짜 / 시간

SQLite 는 전용 DATE 타입 없음 — TEXT (ISO 8601) / INTEGER (unix) / REAL (Julian Day).

```sql
created_at TEXT NOT NULL DEFAULT (datetime('now'))
-- "2026-05-13 14:00:00"

-- subsec (3.42+)
DEFAULT (datetime('now', 'subsec'))
-- "2026-05-13 14:00:00.123"
```

날짜 함수:

```sql
SELECT date('now');                    -- "2026-05-13"
SELECT datetime('now');                 -- "2026-05-13 14:00:00"
SELECT datetime('now', '+1 day');
SELECT datetime('now', 'localtime');
SELECT strftime('%Y-%m', 'now');
SELECT unixepoch('now');                -- integer seconds (3.38+)
SELECT julianday('now');
```

### 5.6 UUID

내장 X → TEXT 또는 BLOB.

```sql
id TEXT PRIMARY KEY              -- "uuid-string"

-- 3.41+ uuid 함수
SELECT lower(hex(randomblob(16)));
-- 또는 어플리케이션에서 v4 / v7
```

### 5.7 JSON (JSON1 — 기본 활성)

```sql
data TEXT NOT NULL CHECK (json_valid(data))

INSERT INTO events (data) VALUES ('{"user":"alice","tags":["a","b"]}');

SELECT json_extract(data, '$.user') FROM events;
SELECT data ->> '$.user' FROM events;     -- 3.38+ 단축

-- 업데이트
UPDATE events SET data = json_set(data, '$.user', 'bob');
UPDATE events SET data = json_insert(data, '$.email', 'a@x.com');
UPDATE events SET data = json_remove(data, '$.tags[0]');

-- 함수
json_object('k','v')
json_array(1,2,3)
json_each, json_tree   -- 테이블 함수
```

### 5.8 JSONB (3.45+, 바이너리)

```sql
data BLOB CHECK (json_valid(data))
INSERT INTO events VALUES (jsonb('{"a":1}'));
SELECT jsonb_extract(data, '$.a') FROM events;
```

→ PG JSONB 처럼 빠름. 새 옵션.

### 5.9 BLOB

```sql
photo BLOB
```

⚠️ 큰 BLOB → DB 비대 + 백업 부담. 파일은 외부 + path 만 권장.

---

## 6. 인덱스 + 함수

### 6.1 Expression Index

```sql
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
WHERE LOWER(email) = ...   -- 인덱스 사용
```

### 6.2 Generated Column

```sql
CREATE TABLE users (
  ...,
  email_domain TEXT AS (substr(email, instr(email,'@')+1)) VIRTUAL,
  -- VIRTUAL (계산만) 또는 STORED (저장)
);

CREATE INDEX idx_email_domain ON users(email_domain);
```

JSON 필드의 인덱스에 유용.

---

## 7. 정렬 (Collation)

| Collation | 의미 |
| --- | --- |
| `BINARY` (기본) | 바이트 비교 |
| `NOCASE` | 대소문자 무시 (ASCII 만) |
| `RTRIM` | 우측 공백 무시 |

```sql
CREATE TABLE users (
  email TEXT COLLATE NOCASE
);

-- 인덱스
CREATE INDEX idx_users_email ON users(email COLLATE NOCASE);
```

⚠️ 한국어 / 유니코드 정렬은 사용자 정의 collation 필요 (라이브러리 단).

---

## 8. NULL

- `IS NULL` / `IS NOT NULL`
- `COALESCE(a, b, c)`
- `IFNULL(a, b)` — 2 개
- `NULLIF(a, b)` — 같으면 NULL

UNIQUE 인덱스에서 NULL 은 여러 개 허용 (SQL 표준).

---

## 9. 정수 PK / ROWID

```sql
CREATE TABLE t (
  id INTEGER PRIMARY KEY,           -- rowid 별칭 — INSERT 시 자동
  ...
);
```

INSERT 시 자동 채워짐. `last_insert_rowid()` 또는 `RETURNING id`.

### 9.1 AUTOINCREMENT (대부분 불필요)

```sql
id INTEGER PRIMARY KEY AUTOINCREMENT
```

ID 재사용 X 보장. 약간 느림.

### 9.2 WITHOUT ROWID

```sql
CREATE TABLE kv (
  k TEXT PRIMARY KEY,
  v TEXT
) WITHOUT ROWID;
```

작은 row + PK 가 자연키일 때 효율 ↑.

---

## 10. Numeric Affinity 의 함정

```sql
CREATE TABLE t (n NUMERIC);
INSERT INTO t VALUES ('1.5');    -- → REAL 1.5
INSERT INTO t VALUES ('hello');   -- → TEXT 'hello' (변환 실패)
```

→ STRICT 로 회피.

---

## 11. 함정

### 11.1 Dynamic Typing 함정
의도와 다른 타입 저장 — STRICT 사용.

### 11.2 `VARCHAR(n)` 의 무의미
길이 제한 X. `TEXT CHECK(length(...))`.

### 11.3 DECIMAL 미지원
돈 / 정확 → 정수 (cent) 또는 TEXT.

### 11.4 Date 의 다양한 표현
TEXT (ISO) 권장. 일관성.

### 11.5 BOOL 의 TINYINT
0/1 — true/false 키워드는 동작하지만 1/0 으로 저장.

### 11.6 NULL 유일성
UNIQUE 컬럼에 여러 NULL 허용.

### 11.7 COLLATE 한국어
기본 X. ICU 컴파일 / 라이브러리 collation 필요.

### 11.8 큰 BLOB
가능하지만 비추 — 파일 시스템 / S3 + URL.

---

## 12. 학습 자료

- **SQLite Datatypes** — sqlite.org/datatype3.html
- **STRICT Tables** — sqlite.org/stricttables.html
- **JSON1 Extension**
- **Date and Time Functions** — sqlite.org/lang_datefunc.html

---

## 13. 관련

- [[sql-syntax]] — SQL
- [[sqlite]] — SQLite hub
