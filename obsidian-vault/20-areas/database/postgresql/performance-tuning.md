---
title: "PostgreSQL 성능 튜닝 — 메모리 / I/O / 쿼리 / autovacuum"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:55:00+09:00
tags:
  - database
  - postgresql
  - performance
  - tuning
---

# PostgreSQL 성능 튜닝

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 메모리 / I/O / autovacuum / 쿼리 / pool |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 튜닝의 순서

1. **올바른 쿼리 / 인덱스** (가장 큰 효과)
2. autovacuum / 통계
3. 메모리 (`shared_buffers`, `work_mem`)
4. WAL / checkpoint
5. Connection pool
6. 하드웨어 / 디스크

> "기본값으로 잘 안 돌면 메모리 / pool 부터, 정상 동작 후엔 쿼리부터."

---

## 2. 메모리

### 2.1 shared_buffers

PostgreSQL 의 공유 캐시. RAM 의 **25%** 권장.

```ini
shared_buffers = 4GB         # 16GB 시스템
```

너무 크면 OS 캐시와 double caching. 너무 작으면 디스크 I/O ↑.

### 2.2 work_mem

각 정렬 / 해시 / GROUP BY 작업당 메모리. **connection × operation** 으로 곱해짐.

```ini
work_mem = 16MB
```

부족하면 `external merge Disk: ...` (디스크로 떨어짐 = 느림).
크면 메모리 폭발 위험. 보통 4MB ~ 64MB.

세션 단위:

```sql
SET work_mem = '256MB';   -- 무거운 쿼리만
SELECT ...;
RESET work_mem;
```

### 2.3 maintenance_work_mem

VACUUM, CREATE INDEX, ALTER TABLE 용. 크게 잡아도 안전 (한 번에 하나).

```ini
maintenance_work_mem = 1GB
```

### 2.4 effective_cache_size

옵티마이저가 추정하는 **OS + Postgres 캐시** 의 총합. 실제 메모리 할당 X — 옵티마이저 힌트.

```ini
effective_cache_size = 12GB   # RAM 의 75%
```

작게 잡으면 Seq Scan 선호 → 인덱스 안 쓸 수 있음.

### 2.5 wal_buffers
보통 자동 (-1). 기본으로 충분.

---

## 3. I/O

### 3.1 random_page_cost

random I/O 의 추정 비용. SSD 면 낮춰야 인덱스 활용 ↑.

```ini
random_page_cost = 1.1    # SSD
random_page_cost = 4.0    # HDD (기본)
```

### 3.2 seq_page_cost

순차 읽기 비용. 보통 1.0.

### 3.3 effective_io_concurrency

비동기 I/O 동시성. SSD = 200, HDD = 1-2.

```ini
effective_io_concurrency = 200
```

---

## 4. WAL / Checkpoint

### 4.1 max_wal_size

checkpoint 사이 WAL 최대. 크게 잡으면 checkpoint 빈도 ↓, 복구 시간 ↑.

```ini
max_wal_size = 4GB
min_wal_size = 1GB
```

### 4.2 checkpoint_completion_target

checkpoint 작업을 얼마나 분산? 0.9 권장.

```ini
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
```

### 4.3 wal_compression

```ini
wal_compression = on    # CPU vs 디스크 트레이드오프 — 보통 on
```

### 4.4 synchronous_commit

`off` 면 응답 ↑ 대신 crash 시 ~200 ms 손실. 일부 워크로드 (이벤트 로그 등) 만.

```ini
synchronous_commit = on   # 기본 — 안전
```

---

## 5. autovacuum

VACUUM 안 돌면 → bloat → 성능 저하. 가장 흔한 운영 사고.

### 5.1 글로벌

```ini
autovacuum = on
autovacuum_naptime = 1min
autovacuum_max_workers = 5
```

### 5.2 임계값

```ini
autovacuum_vacuum_scale_factor = 0.1    # 10% 변경 시 VACUUM
autovacuum_analyze_scale_factor = 0.05  # 5% 변경 시 ANALYZE
autovacuum_vacuum_threshold = 50        # 최소 행 수
```

큰 테이블은 10% = 매우 많은 행. 테이블별로 조정:

```sql
ALTER TABLE big_table SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.005
);
```

### 5.3 비용 제한

```ini
autovacuum_vacuum_cost_limit = 2000   # 기본 200 — 부족
autovacuum_vacuum_cost_delay = 2ms
```

기본은 너무 느림. 운영 환경은 cost_limit 을 2000-5000 으로.

### 5.4 모니터링

```sql
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;
```

자세히 → [[transactions-mvcc]]

---

## 6. 통계 / 옵티마이저

### 6.1 ANALYZE

```sql
ANALYZE users;
ANALYZE (VERBOSE);
```

통계 없으면 옵티마이저는 추측. 대량 INSERT 후 수동 ANALYZE.

### 6.2 default_statistics_target

ANALYZE 표본 크기 (기본 100). 크면 정확 ↑, 시간 ↑.

```ini
default_statistics_target = 100
```

특정 컬럼만:

```sql
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
```

### 6.3 Extended Statistics — 다중 컬럼 상관

```sql
CREATE STATISTICS s_orders ON user_id, status FROM orders;
ANALYZE orders;
```

`user_id` 와 `status` 가 독립이 아닐 때 (예: 한 사용자가 한 상태) — 옵티마이저가 더 정확.

---

## 7. Connection Pool

PG 1 connection = process + ~10 MB. 1000 connection = 10 GB + 컨텍스트 스위치.

### 7.1 PgBouncer

```ini
[databases]
app = host=localhost dbname=app

[pgbouncer]
pool_mode = transaction       # session / transaction / statement
max_client_conn = 1000
default_pool_size = 25
```

| 모드 | 의미 | 호환성 |
| --- | --- | --- |
| `session` | 연결 종료 시 release | 안전, pool 효과 ↓ |
| `transaction` | 트랜잭션 종료 시 release | 표준 권장 |
| `statement` | 매 statement 끝 | 트랜잭션 X |

⚠️ `transaction` 모드에서는 prepared statement / SET / advisory lock 주의.

### 7.2 PgCat

PgBouncer 대안 (Rust, 로드 밸런싱, 샤딩 지원).

---

## 8. JIT (Just-In-Time)

PG 11+ 부터 LLVM JIT. 복잡한 쿼리 가속.

```ini
jit = on
jit_above_cost = 100000
jit_optimize_above_cost = 500000
jit_inline_above_cost = 500000
```

OLTP 짧은 쿼리 → JIT 가 오히려 부담. `jit_above_cost` 를 매우 높이거나 `jit = off`.

---

## 9. 병렬 쿼리

```ini
max_parallel_workers = 8
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
```

큰 Seq Scan / JOIN / Aggregate 자동 병렬.

---

## 10. 쿼리 튜닝 체크리스트

1. **EXPLAIN ANALYZE** 결과 보기
2. Seq Scan 대신 Index Scan 가능?
3. Filter 조건 인덱스 가능?
4. JOIN 방식 적절?
5. Sort 가 메모리 안에서?
6. CTE 가 fence 가 되어 있지 않은가?
7. `NOT IN` → `NOT EXISTS`
8. `OFFSET` 페이지네이션 → keyset
9. 함수 호출이 매 행마다? — `STABLE` / `IMMUTABLE`
10. JSON 경로 인덱스 가능?

자세히 → [[explain-analyze]]

---

## 11. Top 쿼리 발굴

```sql
SELECT
  substring(query, 1, 80) AS q,
  calls,
  ROUND(total_exec_time::numeric, 1) AS total_ms,
  ROUND(mean_exec_time::numeric, 1) AS mean_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

→ 가장 시간 잡아먹는 쿼리 = 튜닝 1순위.

---

## 12. 데드락 / 락 진단

```sql
SELECT
  blocked.pid AS blocked_pid,
  blocked.query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  blocking.usename
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

자세히 → [[transactions-mvcc]]

---

## 13. 디스크 / OS

### 13.1 파일 시스템
- **ext4 / xfs** — 표준
- `noatime` 마운트
- `barrier=1` 유지 (안전)
- ZFS — 좋지만 `full_page_writes = off` + ZFS atomic 필요

### 13.2 swap
PG 가 swap 으로 떨어지면 사망. `vm.swappiness = 1`.

### 13.3 transparent huge pages

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

OLTP 에 안 좋음. 끄는 것을 권장.

### 13.4 NUMA

```bash
numactl --interleave=all postgres ...
```

---

## 14. 모니터링 지표

| 지표 | 좋은 값 | 도구 |
| --- | --- | --- |
| 캐시 적중률 | > 99% | `pg_stat_database.blks_hit / (blks_hit + blks_read)` |
| Bloat | < 20% | `pgstattuple` |
| Replication lag | < 1s | `pg_stat_replication` |
| Lock 대기 | 거의 0 | `pg_locks` |
| Connection 사용률 | < 80% | `pg_stat_activity` |
| autovacuum 활동 | 활발 | `pg_stat_user_tables` |
| WAL 생성 속도 | 안정 | `pg_current_wal_lsn` 차분 |

---

## 15. PgTune — 시작점

[pgtune.leopard.in.ua](https://pgtune.leopard.in.ua) — 자동 추천.

대부분의 운영 환경은 PgTune + 측정 + 미세조정으로 충분.

---

## 16. 함정

### 함정 1 — `work_mem` × N
connection × 동시 operation × work_mem = 메모리 폭발.

### 함정 2 — `shared_buffers` 너무 큼
RAM 의 25% 넘으면 OS 캐시 압박. > 50% 는 거의 항상 손해.

### 함정 3 — autovacuum 비활성
"느려서 끔" → 한 달 뒤 사망. 끄면 안 됨, **튜닝**.

### 함정 4 — `synchronous_commit = off` 의 무지
실시간 시스템에선 데이터 손실 사고.

### 함정 5 — 통계 갱신 누락
대량 INSERT 후 ANALYZE 안 함 → 옵티마이저 오판.

### 함정 6 — Connection Pool 없음
1000 connection 직결 → 메모리 / context switch 폭발. PgBouncer 도입.

### 함정 7 — JIT 의 부작용
짧은 OLTP 쿼리에 JIT 가 오히려 느림. 측정 후 결정.

### 함정 8 — `wal_level = minimal` 의 함정
replication / archive 불가. 사실상 `replica` 가 표준.

---

## 17. 학습 자료

- **PostgreSQL 14 Internals** — Egor Rogov (필독)
- **PostgreSQL High Performance** — Schönig
- **PgTune** — pgtune.leopard.in.ua
- **2ndQuadrant / EDB Blog**
- **Cybertec PostgreSQL Blog** — cybertec-postgresql.com/en

---

## 18. 관련

- [[configuration]] — 파라미터
- [[indexes]] — 인덱스
- [[explain-analyze]] — 쿼리 진단
- [[transactions-mvcc]] — VACUUM / Lock
- [[postgresql]] — PostgreSQL hub
