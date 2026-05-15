---
title: "Database performance — Postgres / MySQL tuning"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:27:00+09:00
tags: [devops, performance, database, postgres, mysql]
---

# Database performance — Postgres / MySQL tuning

**[[performance|↑ performance]]**

---

## 1. DB 가 거의 항상 bottleneck

```
application 의 latency 분석:
  60%+ 가 DB query
  20% 가 외부 API
  20% 가 application logic

→ DB 튜닝 = ROI 가장 높음.
```

---

## 2. 진단 단계 (Postgres)

```sql
-- 1. slow query log
ALTER SYSTEM SET log_min_duration_statement = 100;   -- 100ms+
SELECT pg_reload_conf();

-- 2. 통계
CREATE EXTENSION pg_stat_statements;

SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- 3. 활성 query
SELECT pid, state, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle';

-- 4. 잠금
SELECT * FROM pg_locks WHERE NOT granted;

-- 5. 인덱스 사용 분석
SELECT relname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

---

## 3. EXPLAIN ANALYZE (★ 가장 중요)

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM orders 
WHERE user_id = 123 AND created_at > '2026-01-01';

-- 출력 읽기:
-- Index Scan using ix_orders_user on orders  (cost=0.43..8.45 rows=1 width=100) (actual time=0.123..0.156 rows=1 loops=1)
--   Index Cond: (user_id = 123)
--   Filter: (created_at > '2026-01-01')
--   Rows Removed by Filter: 100   ← ★ 인덱스 부족
-- Planning Time: 0.234 ms
-- Execution Time: 0.789 ms

-- 좋은 신호:
-- Index Scan / Index Only Scan
-- Bitmap Heap Scan + Bitmap Index Scan (대량)
-- 낮은 actual time

-- 나쁜 신호:
-- Seq Scan on big table (★ index 필요)
-- Nested Loop on big tables
-- Filter 의 "Rows Removed by Filter" 큼
-- Sort 가 disk (Sort Method: external merge Disk)
-- 큰 cost / actual time 차이
```

---

## 4. index 디자인

```sql
-- 가장 흔한 — B-tree
CREATE INDEX ix_orders_user_created 
ON orders(user_id, created_at DESC);

-- partial (조건 매칭만)
CREATE INDEX ix_orders_pending 
ON orders(user_id) 
WHERE status = 'pending';

-- covering (Index Only Scan)
CREATE INDEX ix_orders_user_covering 
ON orders(user_id) INCLUDE (status, amount);

-- expression
CREATE INDEX ix_users_email_lower 
ON users (LOWER(email));

-- GIN (JSON / array / full-text)
CREATE INDEX ix_users_meta_gin 
ON users USING gin (meta);

-- BRIN (시간 / 큰 ordered data)
CREATE INDEX ix_events_time_brin 
ON events USING brin (created_at);
```

---

## 5. 흔한 잘못된 index

```sql
-- ❌ 매번 만드는 type 의 column
CREATE INDEX ON users(country);   -- 5개 값 → 거의 의미 없음

-- ❌ leading column 우선순위 잘못
CREATE INDEX ON orders(created_at, user_id);
-- query: WHERE user_id = 123  → index 안 씀

-- ❌ 너무 많은 index
-- 모든 column 마다 → write 느려짐

-- ✅ 자주 같이 쓰는 column 의 composite
CREATE INDEX ON orders(user_id, created_at);
```

---

## 6. N+1 query

```sql
-- ❌ N+1
SELECT * FROM users;
-- → 100 user 각각
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
...

-- ✅ JOIN 또는 IN
SELECT u.*, o.* 
FROM users u 
LEFT JOIN orders o ON o.user_id = u.id;

-- 또는
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 100);
```

```java
// JPA: @EntityGraph 또는 JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

---

## 7. connection pool

```yaml
# HikariCP (Spring Boot 기본)
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # DB max connection 와 협의
      minimum-idle: 5
      connection-timeout: 3000      # 3s 대기
      idle-timeout: 600000          # 10min idle 후 close
      max-lifetime: 1800000         # 30min 후 강제 close
      leak-detection-threshold: 60000   # 1min 안 returned = warn
      pool-name: hikari-main
```

→ 일반: `min((core × 2) + effective_spindles, db_max_connections / instance_count)`.

→ 자주: pod 10 × pool 20 = 200 connection. DB max = 200 → 빠듯.

---

## 8. transaction 길이

```java
// ❌ 긴 transaction
@Transactional
public void process() {
    Order order = repo.find(...);    // DB
    callExternalApi();                // 외부 API (느림)
    order.update();                   // DB
    repo.save(order);
}
// → API 시간 동안 connection 점유

// ✅ 짧게 분리
public void process() {
    Order order = service.load(orderId);   // tx 1
    OrderResult result = externalApi();    // tx 없음
    service.save(order, result);           // tx 2
}
```

→ 외부 API / 큰 계산은 transaction 밖.

---

## 9. read replica 활용

```
master:    write 만
replica:   read (대부분 traffic)

→ load 분산 + master 보호
```

```yaml
# Spring 의 routing DataSource
spring:
  datasource:
    write:
      url: jdbc:postgresql://master/db
    read:
      url: jdbc:postgresql://replica/db
```

```java
@Transactional(readOnly = true)  // ★ 자동 replica routing
public Order find(Long id) { ... }

@Transactional               // master
public Order update(...) { ... }
```

→ replication lag 인식 (write 직후 read 의 stale).

---

## 10. partition (큰 table)

```sql
-- range partitioning (시간)
CREATE TABLE events (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    payload JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- 자동 partition 관리: pg_partman
```

→ 옛 partition 만 archive / drop 가능.

---

## 11. cache

```
Application 수준:
  - Caffeine (local, fast)
  - Redis (distributed)
  
DB 수준:
  - materialized view
  - shared_buffers (PG memory cache)
  - query result cache
```

```sql
-- materialized view
CREATE MATERIALIZED VIEW user_summary AS
SELECT user_id, COUNT(*), SUM(amount)
FROM orders
GROUP BY user_id;

REFRESH MATERIALIZED VIEW CONCURRENTLY user_summary;
```

---

## 12. Postgres 설정 (★)

```
# postgresql.conf

# memory
shared_buffers = 25% of RAM         # 8GB RAM → 2GB
effective_cache_size = 75% of RAM   # 8GB → 6GB
work_mem = RAM / max_connections / 4   # sort / hash 별

# WAL
wal_level = replica
max_wal_size = 4GB
checkpoint_timeout = 10min

# parallel
max_parallel_workers = 8
max_parallel_workers_per_gather = 4

# logging
log_min_duration_statement = 100   # ms

# autovacuum
autovacuum = on
autovacuum_max_workers = 4
```

→ pgtune.leopard.in.ua 에서 자동 권장.

---

## 13. lock contention

```sql
-- 누가 lock 잡고 있나
SELECT
    a.pid as blocker_pid,
    a.usename,
    a.query as blocker_query,
    a.state,
    b.pid as blocked_pid,
    b.query as blocked_query
FROM pg_stat_activity a
JOIN pg_stat_activity b ON b.wait_event_type = 'Lock'
WHERE a.pid != b.pid;

-- 강제 cancel (★ 위험)
SELECT pg_cancel_backend(<pid>);
SELECT pg_terminate_backend(<pid>);
```

```sql
-- row-level lock
SELECT * FROM orders WHERE id = 1 FOR UPDATE;        -- 다른 tx 대기
SELECT * FROM orders WHERE id = 1 FOR UPDATE SKIP LOCKED;  -- queue 패턴
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;       -- 즉시 fail
```

---

## 14. monitoring

```
도구:
  - pg_stat_statements (built-in)
  - pgAnalyze, pganalyze.com
  - DataDog DBM
  - New Relic
  - PMM (Percona)
  - pgBadger (log 분석)
  
metric:
  - active connections
  - replication lag
  - cache hit ratio (> 99%)
  - dead tuples (vacuum 필요)
  - long running query
  - lock wait
```

---

## 15. 함정

1. **EXPLAIN 안 보고 query 작성** — 무작정 production.
2. **index 너무 많음** — write 부담.
3. **N+1** — 가장 흔한.
4. **connection pool 너무 큼** — DB connection 폭주.
5. **transaction 안에 외부 API** — connection 점유.
6. **`SELECT *`** — 큰 row, network 낭비.
7. **autovacuum 끔** — dead tuple 누적 → bloat.
8. **shared_buffers 부족** — disk IO 많음.

---

## 16. 관련

- [[performance|↑ performance]]
- [[caching-strategies]]
- [[../../60-recipes/spring/database/database|↗ DB recipe]]
