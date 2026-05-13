---
title: "PostgreSQL 인덱스 — B-Tree / Hash / GIN / GiST / BRIN"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:25:00+09:00
tags:
  - database
  - postgresql
  - index
---

# PostgreSQL 인덱스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 6 종 인덱스 + Partial / Expression / Covering |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 한 줄

PostgreSQL 인덱스는 **6 가지 타입** + **Partial / Expression / Covering / Unique** 옵션.
타입별로 잘하는 일이 다름 — 어떤 쿼리를 빠르게 할지에 따라 선택.

---

## 2. 인덱스 종류 한눈에

| 종류 | 잘 하는 일 | 예 |
| --- | --- | --- |
| **B-Tree** (기본) | `=, <, >, BETWEEN`, ORDER BY, UNIQUE | 거의 모든 컬럼 |
| **Hash** | `=` 만, 빠른 동등 비교 | UUID, hash |
| **GIN** | 다값 컬럼 (Array, JSONB, full-text) | tags, JSONB |
| **GiST** | 기하 / 범위 / 유사도 | geometry, tsvector, range |
| **SP-GiST** | 비균형 / 공간 분할 | IP, prefix |
| **BRIN** | 거대 테이블 + 정렬된 데이터 | 시계열, 로그 |

---

## 3. B-Tree — 기본

```sql
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
```

### 3.1 잘 하는 일

- `WHERE email = 'a@x.com'`
- `WHERE age BETWEEN 18 AND 30`
- `ORDER BY created_at DESC`
- `WHERE email LIKE 'a%'` (앞쪽 매칭만)
- UNIQUE 제약 백킹

### 3.2 복합 인덱스 — 컬럼 순서가 중요

```sql
CREATE INDEX idx_orders ON orders (user_id, created_at);

-- ✅ 인덱스 활용
WHERE user_id = 1
WHERE user_id = 1 AND created_at > '...'
WHERE user_id = 1 ORDER BY created_at

-- ❌ 인덱스 안 탐 (또는 일부만)
WHERE created_at > '...'
```

원칙: **선택도 (selectivity)** 높은 컬럼 먼저, 또는 등호 → 범위 순.

### 3.3 LIKE 와 인덱스

```sql
-- ✅ 인덱스 사용 — 앞 고정
WHERE email LIKE 'alice%'

-- ❌ 인덱스 사용 X — 뒤 매칭
WHERE email LIKE '%example.com'

-- 해결책: pg_trgm + GIN
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN (email gin_trgm_ops);
WHERE email LIKE '%example.com'   -- 이제 인덱스 사용
```

---

## 4. Hash

```sql
CREATE INDEX idx_users_uuid_hash ON users USING HASH (uuid);
```

- `=` 만 지원 (no `<`, `>`, `BETWEEN`, `ORDER BY`)
- PG 10+ 부터 WAL 로깅 → 안정적
- 대부분의 경우 B-Tree 가 비슷하거나 더 좋음 → **거의 안 씀**

---

## 5. GIN (Generalized Inverted Index)

**다값 컬럼** 의 역색인 (inverted index).

### 5.1 JSONB

```sql
CREATE INDEX idx_events_data ON events USING GIN (data);

-- 활용
WHERE data @> '{"user":"alice"}'
WHERE data ? 'user'
WHERE data ?| ARRAY['user', 'admin']
```

옵션: `jsonb_path_ops` — 더 작고 빠르지만 `@>` 만.

```sql
CREATE INDEX ON events USING GIN (data jsonb_path_ops);
```

### 5.2 Array

```sql
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

WHERE tags @> ARRAY['postgres']
WHERE tags && ARRAY['rust','go']      -- 교집합
```

### 5.3 Full-text Search

```sql
ALTER TABLE articles ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
  ) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search);

SELECT * FROM articles WHERE search @@ to_tsquery('english', 'postgres & tutorial');
```

### 5.4 pg_trgm (trigram)

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);

-- 부분 / 유사도 검색
WHERE name ILIKE '%alice%'
WHERE name % 'alicia'    -- 유사도
ORDER BY name <-> 'alicia'  -- 가장 비슷한 순
```

---

## 6. GiST (Generalized Search Tree)

**범위 / 거리 / 유사도** 등 비-스칼라 비교.

### 6.1 Range / EXCLUDE

```sql
CREATE TABLE reservations (
    id BIGSERIAL PRIMARY KEY,
    room_id INTEGER NOT NULL,
    during  TSTZRANGE NOT NULL,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);
```

### 6.2 지리 (PostGIS)

```sql
CREATE INDEX idx_places_geom ON places USING GIST (geom);

SELECT * FROM places
WHERE ST_DWithin(geom, ST_MakePoint(127.0, 37.5), 1000);
```

### 6.3 nearest neighbor

```sql
SELECT * FROM places
ORDER BY geom <-> ST_MakePoint(127.0, 37.5)
LIMIT 10;
```

---

## 7. BRIN (Block Range Index)

거대 테이블 + 데이터가 **물리 순서로 정렬** 되어 있을 때.

```sql
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at);
```

- 인덱스 크기 매우 작음 (B-Tree 의 0.1% 수준)
- `created_at` 순으로 append 만 되는 로그 / 시계열에 이상적
- 정확도는 낮음 — block 범위 단위로 필터

옵션:

```sql
USING BRIN (created_at) WITH (pages_per_range = 32)
```

---

## 8. SP-GiST

비균형 트리 (Space-Partitioned).

```sql
CREATE INDEX ON ips USING SPGIST (ip);  -- INET 의 prefix 검색
```

흔히 안 씀. 특화된 케이스만.

---

## 9. Partial Index (부분 인덱스)

**조건을 만족하는 행만** 인덱스.

```sql
-- '활성' 사용자만 인덱스
CREATE INDEX idx_users_active ON users (email)
WHERE active = true;

-- 미완료 주문만
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';
```

### 9.1 효과
- 인덱스 크기 ↓↓ (활성 비율이 작으면)
- 인덱스 갱신 비용 ↓
- 쿼리에 동일 WHERE 가 있어야 사용됨

---

## 10. Expression Index (표현식 인덱스)

**계산된 값** 에 인덱스.

```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

WHERE LOWER(email) = 'alice@x.com'   -- ✅ 사용
```

### 10.1 JSON 경로

```sql
CREATE INDEX ON events ((data->>'user'));
WHERE data->>'user' = 'alice'
```

### 10.2 Generated column 대안

PG 12+ `GENERATED ... STORED` 컬럼 + 일반 인덱스 가 더 깔끔할 수도.

---

## 11. Unique Index

```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);

-- partial + unique — 활성 사용자 중에서만 email 유일
CREATE UNIQUE INDEX idx_users_email_active
ON users (email) WHERE active;
```

---

## 12. Covering Index (INCLUDE)

쿼리에 필요한 컬럼을 인덱스에 **포함** (검색은 안 하지만 같이 저장) — Index Only Scan.

```sql
CREATE INDEX idx_orders_user_incl
ON orders (user_id) INCLUDE (created_at, total);

-- 이 쿼리는 테이블 접근 없이 인덱스만 읽음
SELECT created_at, total FROM orders WHERE user_id = 1;
```

---

## 13. CONCURRENTLY — 락 없이 인덱스 생성

```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

- 운영 중 인덱스 생성 시 필수
- 단점: 트랜잭션 안에서 X, 더 느림, 실패 시 INVALID 인덱스 남음 (DROP 후 재시도)

```sql
-- 실패한 invalid 인덱스 찾기
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

---

## 14. 인덱스 진단

### 14.1 사용 통계

```sql
SELECT
  schemaname, relname AS table, indexrelname AS index,
  idx_scan, idx_tup_read, idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;   -- 안 쓰는 인덱스 발견
```

### 14.2 안 쓰는 인덱스
`idx_scan = 0` 이면 후보. PK / UNIQUE 는 제외하고 검토.

### 14.3 중복 인덱스

```sql
SELECT a.indexrelid::regclass, b.indexrelid::regclass
FROM pg_index a, pg_index b
WHERE a.indrelid = b.indrelid
  AND a.indkey::text = b.indkey::text
  AND a.indexrelid < b.indexrelid;
```

---

## 15. REINDEX — 재구축

```sql
REINDEX TABLE users;
REINDEX INDEX idx_users_email;
REINDEX TABLE CONCURRENTLY users;   -- PG 12+
REINDEX DATABASE app;
```

언제? — Bloat 심한 경우, 인덱스 손상, B-Tree 비효율.

---

## 16. 함정

### 함정 1 — 너무 많은 인덱스
INSERT / UPDATE 시 모든 인덱스 갱신. 인덱스 5+ 면 검토.

### 함정 2 — 작은 테이블에 인덱스
< 1000 행은 seq scan 이 더 빠를 수도. 옵티마이저가 무시.

### 함정 3 — Functional dependency 에 의존
`LOWER(email)` 비교한다면 인덱스도 `LOWER(email)` 표현식으로.

### 함정 4 — 복합 인덱스 컬럼 순서
첫 컬럼이 WHERE 에 없으면 활용 X.

### 함정 5 — `IS NULL` 의 인덱스
B-Tree 도 `IS NULL` 검색 가능 — Partial Index 도 고려.

### 함정 6 — pg_trgm 없이 `LIKE '%word%'`
정규 B-Tree 로는 인덱스 안 탐. pg_trgm + GIN 필수.

### 함정 7 — `CONCURRENTLY` 의 실패
INVALID 인덱스 남음 — DROP 후 재시도.

---

## 17. 관련

- [[explain-analyze]] — 인덱스가 실제로 사용되는지 확인
- [[performance-tuning]] — 쿼리 튜닝
- [[postgresql]] — PostgreSQL hub
