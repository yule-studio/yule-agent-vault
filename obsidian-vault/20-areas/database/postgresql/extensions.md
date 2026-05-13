---
title: "PostgreSQL Extensions — PostGIS / pgvector / 등"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:45:00+09:00
tags:
  - database
  - postgresql
  - extension
---

# PostgreSQL Extensions

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 주요 extension 카탈로그 |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. Extension 메커니즘

PostgreSQL 의 핵심 가치 중 하나 — **확장 가능한 아키텍처**.

```sql
CREATE EXTENSION extname;
CREATE EXTENSION IF NOT EXISTS extname VERSION '1.0';
ALTER EXTENSION extname UPDATE TO '1.1';
DROP EXTENSION extname;

SELECT * FROM pg_available_extensions;
SELECT * FROM pg_extension;
```

설치 위치: `$PGSHARE/extension/`.

---

## 2. 데이터 / 인덱스 확장

### 2.1 pg_trgm — Trigram 유사도

```sql
CREATE EXTENSION pg_trgm;

-- LIKE '%word%' 인덱스 가능
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);

-- 유사도
SELECT * FROM users WHERE name % 'alice';
SELECT name, similarity(name, 'alice') FROM users ORDER BY name <-> 'alice';
```

### 2.2 unaccent — 발음 부호 제거

```sql
CREATE EXTENSION unaccent;
SELECT unaccent('café');  -- 'cafe'
```

### 2.3 hstore — Key-Value

```sql
CREATE EXTENSION hstore;
SELECT 'a => 1, b => 2'::hstore;
-- JSONB 가 더 강력 — 레거시
```

### 2.4 btree_gin / btree_gist
스칼라 컬럼을 GIN/GiST 와 함께 복합 인덱스에 포함.

```sql
CREATE EXTENSION btree_gist;
CREATE INDEX ON reservations USING GIST (room_id, during);
```

### 2.5 citext — 대소문자 무시 텍스트

```sql
CREATE EXTENSION citext;
CREATE TABLE users (email CITEXT UNIQUE);
-- 'a@x.com' == 'A@X.COM'
```

---

## 3. PostGIS — 지리정보

```sql
CREATE EXTENSION postgis;

CREATE TABLE places (
    id BIGSERIAL PRIMARY KEY,
    name TEXT,
    geom GEOGRAPHY(POINT, 4326)
);

INSERT INTO places (name, geom) VALUES
('서울', ST_MakePoint(127.0, 37.5)::geography);

-- 가까운 곳 찾기
SELECT name, ST_DistanceSphere(geom::geometry, ST_MakePoint(127.0, 37.5))
FROM places
ORDER BY geom <-> ST_MakePoint(127.0, 37.5)::geography
LIMIT 10;
```

### 3.1 자주 쓰는 함수
- `ST_MakePoint(lon, lat)`
- `ST_Distance`, `ST_DistanceSphere`, `ST_DWithin`
- `ST_Contains`, `ST_Intersects`
- `ST_Buffer`, `ST_Union`
- `ST_AsGeoJSON`

### 3.2 인덱스 — GiST

```sql
CREATE INDEX idx_places_geom ON places USING GIST (geom);
```

---

## 4. pgvector — 벡터 / AI

```sql
CREATE EXTENSION vector;

CREATE TABLE embeddings (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536)   -- OpenAI ada-002 차원
);

-- nearest neighbor
SELECT content, embedding <=> '[0.1, 0.2, ...]' AS distance
FROM embeddings
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 10;
```

### 4.1 거리 연산자
- `<->` Euclidean (L2)
- `<#>` Inner product
- `<=>` Cosine

### 4.2 인덱스 — IVFFlat / HNSW

```sql
-- HNSW (PG 16+)
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops);

-- IVFFlat
CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

---

## 5. TimescaleDB — 시계열

PostgreSQL extension. 거대 시계열 데이터에 hypertable, continuous aggregate, retention.

```sql
CREATE EXTENSION timescaledb;

CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device TEXT,
    value DOUBLE PRECISION
);
SELECT create_hypertable('metrics', 'time');

-- 압축
ALTER TABLE metrics SET (timescaledb.compress);
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- 자동 retention
SELECT add_retention_policy('metrics', INTERVAL '90 days');

-- Continuous aggregate
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket, device, AVG(value)
FROM metrics
GROUP BY bucket, device;
```

---

## 6. Citus — 분산 / 샤딩

다중 노드 분산 PostgreSQL. Microsoft 가 인수, Azure Hyperscale.

```sql
CREATE EXTENSION citus;

SELECT create_distributed_table('events', 'user_id');
-- user_id 로 샤딩
```

OLAP / HTAP / 멀티테넌시.

---

## 7. 모니터링 / 통계

### 7.1 pg_stat_statements (필수)

```ini
shared_preload_libraries = 'pg_stat_statements'
```

```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;

-- 초기화
SELECT pg_stat_statements_reset();
```

### 7.2 auto_explain

```ini
shared_preload_libraries = 'auto_explain,pg_stat_statements'
auto_explain.log_min_duration = 500ms
auto_explain.log_analyze = on
auto_explain.log_buffers = on
```

### 7.3 pg_buffercache

```sql
CREATE EXTENSION pg_buffercache;

-- shared_buffers 안에 무엇이?
SELECT c.relname, count(*) * 8 AS kb
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY kb DESC LIMIT 10;
```

### 7.4 pgstattuple

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('users');   -- bloat 검사
SELECT * FROM pgstatindex('idx_users_email');
```

---

## 8. UUID / ID

### 8.1 pgcrypto

```sql
CREATE EXTENSION pgcrypto;
SELECT gen_random_uuid();       -- UUID v4
SELECT crypt('password', gen_salt('bf'));  -- bcrypt
```

### 8.2 uuid-ossp

```sql
CREATE EXTENSION "uuid-ossp";
SELECT uuid_generate_v4();
SELECT uuid_generate_v1();    -- 시간 기반 (개인정보 노출 위험)
```

---

## 9. 외부 데이터 — Foreign Data Wrapper (FDW)

### 9.1 postgres_fdw — 다른 PG

```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER remote_pg FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote.example.com', dbname 'app');

CREATE USER MAPPING FOR app SERVER remote_pg
OPTIONS (user 'app', password '...');

IMPORT FOREIGN SCHEMA public FROM SERVER remote_pg INTO remote;

SELECT * FROM remote.users;
```

### 9.2 file_fdw — CSV

```sql
CREATE EXTENSION file_fdw;
CREATE FOREIGN TABLE csv_data (a INT, b TEXT)
SERVER file
OPTIONS (filename '/tmp/data.csv', format 'csv');
```

### 9.3 외부 다양

- `mongo_fdw` — MongoDB
- `redis_fdw` — Redis
- `mysql_fdw` — MySQL
- `parquet_fdw` — Parquet (분석)

---

## 10. 보안

### 10.1 pgaudit

세션 / 객체 단위 감사 로그.

```ini
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
```

### 10.2 anon (PostgreSQL Anonymizer)

PII 마스킹.

---

## 11. 기타 유용

### 11.1 plpgsql_check
PL/pgSQL 정적 분석.

### 11.2 plv8 / plpython3u
JavaScript / Python 으로 함수 작성.

### 11.3 cron / pg_cron

```sql
CREATE EXTENSION pg_cron;

SELECT cron.schedule('cleanup', '0 3 * * *',
  $$DELETE FROM logs WHERE created_at < now() - INTERVAL '90 days'$$);
```

### 11.4 pg_partman
파티션 자동 관리.

### 11.5 wal2json
WAL → JSON (CDC, Debezium).

---

## 12. 설치 경로

```bash
# OS 패키지 — 일반적
apt install postgresql-16-postgis-3
apt install postgresql-16-pgvector

# PGDG repo 권장
# extensions 디렉터리: pg_config --sharedir
```

### 12.1 관리형 서비스 제약
RDS / Cloud SQL 은 사전 승인된 extension 만. 사용 가능 목록 확인 필수.

---

## 13. 함정

### 함정 1 — shared_preload_libraries 변경
재시작 필요. 미리 계획.

### 함정 2 — extension 의존성
`PostGIS` 는 `postgis` 외 여러 서브-extension. `CREATE EXTENSION postgis;` 가 모두 처리.

### 함정 3 — Major upgrade
extension 도 같이 업데이트 필요 (`ALTER EXTENSION ... UPDATE`).

### 함정 4 — 신뢰할 수 없는 언어
`plpython3u`, `plperlu` 는 슈퍼유저만. 보안 고려.

### 함정 5 — pg_stat_statements 의 메모리
`pg_stat_statements.max` 기본 5000. 너무 크면 메모리 ↑.

---

## 14. 학습 자료

- **PGXN** — pgxn.org (extension 카탈로그)
- **PostgreSQL Extensions Documentation**
- **PostGIS Documentation** — postgis.net/docs

---

## 15. 관련

- [[configuration]] — shared_preload_libraries
- [[performance-tuning]] — 진단 extension
- [[postgresql]] — PostgreSQL hub
