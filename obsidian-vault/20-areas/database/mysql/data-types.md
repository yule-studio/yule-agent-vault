---
title: "MySQL 데이터 타입"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:25:00+09:00
tags:
  - database
  - mysql
  - data-types
---

# MySQL 데이터 타입

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 숫자 / 문자 / 날짜 / JSON |

**[[mysql|↑ MySQL hub]]**

---

## 1. 숫자

| 타입 | byte | 범위 |
| --- | --- | --- |
| `TINYINT` | 1 | -128 ~ 127 (UNSIGNED 0 ~ 255) |
| `SMALLINT` | 2 | ±32K |
| `MEDIUMINT` | 3 | ±8M |
| `INT` | 4 | ±2.1B |
| `BIGINT` | 8 | ±9.2 × 10^18 |
| `DECIMAL(p, s)` | 가변 | 임의 정밀도 (돈) |
| `FLOAT` | 4 | 단정밀도 |
| `DOUBLE` | 8 | 배정밀도 |

### 1.1 UNSIGNED

```sql
id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

음수 불필요 시 UNSIGNED → 범위 2배. PK 표준.

### 1.2 DECIMAL — 돈

```sql
amount DECIMAL(12, 2) NOT NULL   -- 99 억 + 소수점 2
```

FLOAT 사용 X (오차).

### 1.3 BOOL / BOOLEAN
`TINYINT(1)` 의 별칭. 진짜 boolean 타입 없음.

---

## 2. 문자

### 2.1 CHAR / VARCHAR

| 타입 | 의미 |
| --- | --- |
| `CHAR(n)` | 고정 길이 n 문자 (padding) |
| `VARCHAR(n)` | 가변, 최대 n 문자 + 1-2 byte 길이 |

```sql
status CHAR(2) NOT NULL     -- 'OK', 'NG'
email  VARCHAR(320) NOT NULL
```

### 2.2 길이는 **문자** 단위 (5.0+)
`VARCHAR(255)` = 255 문자. utf8mb4 면 행당 최대 1020 byte.

### 2.3 TEXT 계열

| 타입 | 최대 |
| --- | --- |
| `TINYTEXT` | 255 byte |
| `TEXT` | 64 KB |
| `MEDIUMTEXT` | 16 MB |
| `LONGTEXT` | 4 GB |

### 2.4 VARCHAR vs TEXT

| 항목 | VARCHAR | TEXT |
| --- | --- | --- |
| 인덱스 길이 제한 | 자유 | 접두만 (`name(20)`) |
| 정렬 / 임시 테이블 | 메모리 | 디스크 |
| 기본값 | 가능 | 가능 (8.0+) |

**보통 VARCHAR(N)** — N 은 충분히 크게. 매우 큰 텍스트만 TEXT/MEDIUMTEXT.

### 2.5 utf8mb4 필수
3 byte `utf8` 은 이모지 X. **항상 `utf8mb4`**.

---

## 3. 날짜 / 시간

| 타입 | byte | 범위 / 의미 |
| --- | --- | --- |
| `DATE` | 3 | `1000-01-01 ~ 9999-12-31` |
| `TIME` | 3 | `-838:59:59 ~ 838:59:59` |
| `DATETIME` | 5-8 | `1000 ~ 9999` (시간대 X) |
| `TIMESTAMP` | 4-7 | `1970 ~ 2038`, **UTC 저장 + 시간대 변환** |
| `YEAR` | 1 | 1901-2155 |

### 3.1 DATETIME vs TIMESTAMP

| 항목 | DATETIME | TIMESTAMP |
| --- | --- | --- |
| 시간대 | 무관 (있는 그대로) | UTC 저장 + 표시 변환 |
| 범위 | 9999 까지 | 2038 |
| 권장 | ✅ 운영 표준 | ⚠️ 2038 함정 |

**DATETIME(6)** + UTC 저장 + 응용에서 변환 권장.

### 3.2 마이크로초

```sql
created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
```
`(6)` 이 마이크로초 (6자리). 1.0+ 부터.

### 3.3 자주 쓰는 함수

```sql
SELECT NOW(), CURDATE(), CURTIME();
SELECT DATE_SUB(NOW(), INTERVAL 7 DAY);
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
SELECT UNIX_TIMESTAMP(NOW());
SELECT FROM_UNIXTIME(1700000000);
SELECT CONVERT_TZ(NOW(), '+00:00', '+09:00');
```

---

## 4. JSON

```sql
CREATE TABLE events (
    id   BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    data JSON NOT NULL
);

INSERT INTO events (data) VALUES
('{"user":"alice","items":[1,2,3]}');

-- 조회
SELECT data->'$.user' FROM events;         -- JSON 값
SELECT data->>'$.user' FROM events;        -- unquote (text)
SELECT JSON_EXTRACT(data, '$.items[0]') FROM events;

-- 조건
WHERE data->>'$.user' = 'alice'
WHERE JSON_CONTAINS(data, '"alice"', '$.user')

-- 업데이트
UPDATE events SET data = JSON_SET(data, '$.user', 'bob') WHERE id=1;
UPDATE events SET data = JSON_ARRAY_APPEND(data, '$.items', 4) WHERE id=1;
```

### 4.1 자주 쓰는 JSON 함수

- `JSON_OBJECT(k, v, k, v)`, `JSON_ARRAY(...)` — 생성
- `JSON_EXTRACT(j, '$.path')` — 추출 (`j->'$.path'`)
- `JSON_UNQUOTE(...)` — string 따옴표 제거 (`j->>'...'`)
- `JSON_SET`, `JSON_INSERT`, `JSON_REPLACE`, `JSON_REMOVE`
- `JSON_CONTAINS`, `JSON_SEARCH`
- `JSON_LENGTH`, `JSON_KEYS`
- `JSON_TABLE(...)` (8.0+) — JSON → 행

### 4.2 인덱스 — Generated Column 으로

```sql
ALTER TABLE events
ADD COLUMN user_name VARCHAR(100)
  AS (data->>'$.user') STORED,
ADD INDEX idx_user_name (user_name);
```

또는 **Functional Index** (8.0.13+):

```sql
ALTER TABLE events ADD INDEX idx_user ((CAST(data->>'$.user' AS CHAR(100))));
```

### 4.3 vs PostgreSQL JSONB
PG JSONB 가 **인덱스 / 연산자** 면에서 더 강력. MySQL 도 8.0+ 면 실용 수준.

---

## 5. ENUM / SET

### 5.1 ENUM

```sql
status ENUM('pending', 'paid', 'shipped', 'cancelled') NOT NULL DEFAULT 'pending'
```

저장은 1-2 byte (인덱스). 값 추가 가능, 삭제는 사실상 안 됨.

⚠️ 운영 변경 잦으면 **lookup 테이블 + FK** 가 더 유연.

### 5.2 SET

```sql
roles SET('admin', 'editor', 'viewer')
```
거의 안 씀 — 정규화 위반.

---

## 6. UUID

내장 UUID 타입 없음 → `CHAR(36)` 또는 `BINARY(16)`.

```sql
id BINARY(16) NOT NULL PRIMARY KEY
-- 입력
INSERT INTO t VALUES (UUID_TO_BIN(UUID()));
-- 출력
SELECT BIN_TO_UUID(id) FROM t;

-- 시간 정렬 (8.0)
SELECT UUID_TO_BIN(UUID(), 1);   -- swap_flag — 인덱스 친화
```

### 6.1 vs INT PK
UUID PK 의 단점: 16 byte (vs 8 byte), random → 인덱스 fragmentation. 큰 분산 환경 외에는 BIGINT 권장.

---

## 7. BLOB

| 타입 | 최대 |
| --- | --- |
| `TINYBLOB` | 255 |
| `BLOB` | 64 KB |
| `MEDIUMBLOB` | 16 MB |
| `LONGBLOB` | 4 GB |

이미지 / 파일은 보통 **S3 / object storage** + URL 만 DB 에 저장.

---

## 8. 공간

```sql
location POINT NOT NULL SRID 4326,
SPATIAL INDEX (location)
```

함수: `ST_Distance_Sphere`, `ST_Contains`, `ST_Within`, ...
PostGIS 만큼은 아니지만 5.7+ 부터 실용.

---

## 9. Generated Columns

### 9.1 종류

```sql
ALTER TABLE users
ADD COLUMN full_name VARCHAR(200)
  AS (CONCAT(first_name, ' ', last_name)) VIRTUAL;

ADD COLUMN email_domain VARCHAR(100)
  AS (SUBSTRING_INDEX(email, '@', -1)) STORED;
```

| 종류 | 의미 |
| --- | --- |
| `VIRTUAL` (기본) | 계산만, 디스크 X. 인덱스 가능 |
| `STORED` | 디스크 저장. 인덱스 가능 |

JSON 인덱싱 / 검색 가속에 유용.

---

## 10. AUTO_INCREMENT

```sql
id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

`innodb_autoinc_lock_mode`:
- `0` — 전통 (lock 길게)
- `1` (기본) — interleaved (대부분)
- `2` — interleaved 항상 (binlog ROW 필요)

---

## 11. NULL

- NULL 은 인덱스에 포함되지만 비교는 `IS NULL`
- `COUNT(col)` 은 NULL 제외, `COUNT(*)` 는 포함
- `UNIQUE` 인덱스에 NULL 은 여러 행 허용 (SQL 표준)

---

## 12. 컬렉션

MySQL 은 **Array 타입 없음** — JSON 또는 별도 테이블.

```sql
-- ❌ MySQL 에 array 없음
-- ✅ JSON 으로
tags JSON,   -- ["a","b","c"]
-- ✅ 별도 테이블
CREATE TABLE post_tags (post_id BIGINT, tag VARCHAR(50), PRIMARY KEY (post_id, tag));
```

---

## 13. 함정

### 함정 1 — `utf8` (3 byte) 사용
이모지 깨짐. **항상 `utf8mb4`**.

### 함정 2 — `TIMESTAMP` 의 2038 한계
DATETIME(6) 권장.

### 함정 3 — `FLOAT` 로 돈 저장
오차. DECIMAL 사용.

### 함정 4 — `VARCHAR(255)` 만 쓰는 관습
의미에 맞게. 짧으면 짧게 — 인덱스 키 길이에 영향.

### 함정 5 — JSON 안에 모두 넣기
정규화 필요한 데이터를 JSON 에 → 인덱스 / 일관성 어려움.

### 함정 6 — ENUM 의존
값 변경이 잦으면 lookup 테이블이 더 유연.

### 함정 7 — UTF-8 collation `_unicode_ci` vs `_0900_ai_ci`
8.0+ 는 `0900_ai_ci` 가 표준 (정렬 정확). 마이그레이션 시 주의.

### 함정 8 — `BOOLEAN` 의 실체
`TINYINT(1)` 별칭. 0/1 외 값도 저장됨.

---

## 14. 관련

- [[sql-syntax]] — SQL
- [[indexes]] — 인덱스
- [[mysql]] — MySQL hub
