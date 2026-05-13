---
title: "MySQL 인덱스 — Clustered / B+ Tree / FULLTEXT / Spatial"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:40:00+09:00
tags:
  - database
  - mysql
  - index
---

# MySQL 인덱스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | B+ Tree / Covering / FULLTEXT / Hash / Spatial |

**[[mysql|↑ MySQL hub]]**

---

## 1. 한 줄

MySQL (InnoDB) 의 인덱스는 **B+ Tree** 가 기본. PK 는 **Clustered**, 나머지는 **Secondary**.
FULLTEXT, Spatial, Hash (MEMORY) 가 보조.

---

## 2. Clustered Index — PK 가 데이터

```
InnoDB:
  PK 의 B+ Tree 리프 = 모든 컬럼 데이터
  → PK 검색 1 트리만
  → PK 가 곧 행의 물리 정렬
```

함의:
- PK 가 작고 단조 증가 → 페이지 채워짐, fragmentation ↓
- PK 가 무작위 (UUIDv4) → 매번 다른 페이지에 INSERT → 비효율
- Secondary 인덱스 = PK 값 보관 → PK 가 크면 모든 인덱스가 큼

자세히 → [[innodb-engine]]

---

## 3. Secondary Index

```
Secondary B+ Tree 리프 = (인덱스 키, PK 값)
→ 인덱스 검색 후 PK 로 다시 검색 (lookup)
→ 또는 Covering 으로 한 번에 끝
```

```sql
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
```

---

## 4. 복합 인덱스 (Multi-column)

**왼쪽 접두** 가 핵심:

```sql
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at);

-- ✅ 사용
WHERE user_id = 1
WHERE user_id = 1 AND created_at > '...'
WHERE user_id = 1 ORDER BY created_at

-- ❌ 안 됨
WHERE created_at > '...'           -- 첫 컬럼 없음
WHERE user_id > 1 AND created_at = '...'   -- 첫 컬럼 범위 후 = 무시
```

원칙: 등호 (`=`) → 범위 (`<,>,BETWEEN`) → 정렬.

---

## 5. Covering Index — Index Only Scan

쿼리에 필요한 모든 컬럼이 인덱스에 있으면 **테이블 접근 없이** 인덱스만 읽기.

```sql
CREATE INDEX idx_orders_covering ON orders (user_id, created_at, status, total);

-- 이 쿼리는 인덱스만 읽음
SELECT created_at, status, total
FROM orders WHERE user_id = 1;
```

EXPLAIN 에서 `Using index` 표시.

⚠️ **MySQL 8.0+ 의 INVISIBLE INDEX, INSTANT DDL** 와 함께 인덱스 설계 자유 ↑.

---

## 6. 인덱스 종류

### 6.1 PRIMARY KEY

```sql
PRIMARY KEY (id)
-- 클러스터드. 유일. NULL 불가.
```

### 6.2 UNIQUE

```sql
UNIQUE KEY uk_users_email (email)
ALTER TABLE users ADD UNIQUE INDEX idx_email (email);
```

### 6.3 일반 (B+ Tree)

```sql
KEY idx_role (role)
INDEX idx_role (role)   -- 동의어
```

### 6.4 FULLTEXT

자세히 → 아래 9 절.

### 6.5 SPATIAL

```sql
SPATIAL INDEX (location)   -- POINT / GEOMETRY
```

### 6.6 HASH
MEMORY 엔진의 기본. InnoDB 는 **Adaptive Hash** 가 자동 생성.

---

## 7. 접두 인덱스 (Prefix Index)

긴 컬럼 (TEXT, 큰 VARCHAR) 의 **앞 N byte** 만 인덱스.

```sql
CREATE INDEX idx_users_email_prefix ON users (email(20));
```

장점: 인덱스 크기 ↓.
단점: ORDER BY / Covering 불가. SELECTIVITY 계산 후 선택.

```sql
-- 적절한 길이 찾기
SELECT COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) FROM users;
SELECT COUNT(DISTINCT LEFT(email, 20)) / COUNT(*) FROM users;
-- 0.95+ 면 충분
```

---

## 8. Functional Index (8.0.13+)

```sql
ALTER TABLE users ADD INDEX idx_email_lower ((LOWER(email)));

WHERE LOWER(email) = 'alice@x.com'   -- ✅ 사용
```

### 8.1 JSON 경로

```sql
ALTER TABLE events
ADD INDEX idx_user ((CAST(data->>'$.user' AS CHAR(100))));

WHERE data->>'$.user' = 'alice'   -- ✅ 사용
```

### 8.2 Generated Column 대안 (이전 버전 호환)

```sql
ALTER TABLE events
ADD COLUMN user_name VARCHAR(100) AS (data->>'$.user') STORED,
ADD INDEX idx_user_name (user_name);
```

---

## 9. FULLTEXT

```sql
ALTER TABLE articles ADD FULLTEXT INDEX idx_search (title, body)
  WITH PARSER ngram;   -- 한국어 / 중국어 / 일본어용 (CJK)

-- 검색
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('postgresql' IN NATURAL LANGUAGE MODE);

-- Boolean 모드
WHERE MATCH(...) AGAINST('+postgres -mysql' IN BOOLEAN MODE);
```

### 9.1 ngram (CJK)

```ini
[mysqld]
ngram_token_size = 2     # 기본 — 한글 등에 권장
```

### 9.2 한계
대규모 / 한국어 형태소 분석 → **Elasticsearch**.

---

## 10. INVISIBLE Index (8.0+)

```sql
ALTER TABLE users ALTER INDEX idx_role INVISIBLE;
-- 옵티마이저가 무시 — 인덱스 영향 시뮬레이션
ALTER TABLE users ALTER INDEX idx_role VISIBLE;
```

드롭 전에 visible / invisible 테스트 → 영향 없으면 DROP.

---

## 11. Descending Index (8.0+)

```sql
CREATE INDEX idx_orders ON orders (user_id ASC, created_at DESC);
-- 정렬 순서 그대로 정렬된 인덱스
```

이전 버전은 무시되어 ASC 와 동일. 8.0+ 부터 실제.

---

## 12. 인덱스 카디널리티 / 통계

```sql
SHOW INDEX FROM users\G

ANALYZE TABLE users;          -- 통계 갱신
```

`Cardinality` = 추정 고유 값 수. 옵티마이저 결정의 핵심.

---

## 13. 사용 / 미사용 인덱스 진단

### 13.1 사용 여부 — Performance Schema

```sql
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;
```

### 13.2 미사용 후보 검토 → INVISIBLE → DROP

---

## 14. 인덱스 권장 / 비권장

### 14.1 ✅ 만들기
- WHERE / JOIN / ORDER BY / GROUP BY 자주 쓰는 컬럼
- FK 컬럼 (CASCADE 성능)
- 카디널리티 높음

### 14.2 ❌ 안 만들기
- 카디널리티 매우 낮음 (예: boolean)
- 자주 변경되는 컬럼
- 너무 긴 컬럼 (prefix index 검토)
- 작은 테이블

---

## 15. CONCURRENT 인덱스 생성

```sql
ALTER TABLE users ADD INDEX idx_name (name), ALGORITHM=INPLACE, LOCK=NONE;
```

- `INSTANT` — 메타데이터만 (컬럼 / 인덱스 추가 일부, 8.0+)
- `INPLACE + LOCK=NONE` — DML 허용
- 안 되면 **pt-online-schema-change / gh-ost**

---

## 16. EXPLAIN 의 인덱스 신호

```
type:     const < eq_ref < ref < range < index < ALL
key:      실제 사용한 인덱스
key_len:  사용한 인덱스 길이
rows:     추정 행 수
Extra:    Using index, Using where, Using filesort, Using temporary
```

자세히 → [[explain]]

---

## 17. 락과 인덱스

인덱스 없는 WHERE → 테이블 락 효과 (모든 행 검사). 항상 적절한 인덱스로 좁히기.

```sql
-- ❌ status 인덱스 없음 → 모든 행 검사 + 락
UPDATE orders SET status = 'cancelled' WHERE status = 'pending';

-- ✅ status 인덱스
```

---

## 18. 함정

### 함정 1 — PK 가 무작위 (UUIDv4)
페이지 분할 폭증. BIGINT AUTO_INCREMENT 또는 UUIDv7.

### 함정 2 — 너무 많은 인덱스
INSERT/UPDATE 시 모두 갱신. 5+ 면 검토.

### 함정 3 — 복합 인덱스 컬럼 순서 잘못
첫 컬럼 없는 WHERE → 인덱스 무시.

### 함정 4 — LIKE '%xxx' 의 인덱스 미사용
앞 와일드카드는 B+ Tree 못 씀. FULLTEXT 또는 역순 컬럼.

### 함정 5 — Implicit Type Conversion
`WHERE varchar_col = 1` → 인덱스 미사용. 정확한 타입.

### 함정 6 — 함수 적용
`WHERE LOWER(email) = ...` → 인덱스 미사용. Functional Index 또는 Generated Column.

### 함정 7 — 작은 테이블 인덱스
< 1000 행은 효과 X.

### 함정 8 — Visible 빼고 DROP
바로 DROP → 롤백 비용. INVISIBLE → 모니터링 → DROP.

---

## 19. 학습 자료

- **High Performance MySQL** Ch. 5 (Indexing)
- **MySQL Reference Manual** Ch. 8.3 (Optimization and Indexes)
- **Percona Blog** — index design

---

## 20. 관련

- [[innodb-engine]] — clustered index
- [[explain]] — 실행 계획
- [[performance-tuning]] — 튜닝
- [[mysql]] — MySQL hub
