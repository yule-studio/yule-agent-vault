---
title: "MySQL 성능 튜닝 — Buffer Pool / Slow Log / 인덱스 / 옵티마이저"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:05:00+09:00
tags:
  - database
  - mysql
  - performance
  - tuning
---

# MySQL 성능 튜닝

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 메모리 / 인덱스 / 쿼리 / 옵티마이저 |

**[[mysql|↑ MySQL hub]]**

---

## 1. 튜닝의 순서

1. **올바른 쿼리 / 인덱스** (가장 큰 효과)
2. Buffer Pool / 메모리
3. 옵티마이저 / 통계
4. 트랜잭션 / 락
5. 디스크 / OS
6. Connection pool

---

## 2. 메모리

### 2.1 innodb_buffer_pool_size (1순위)

```ini
innodb_buffer_pool_size = 12G    # RAM 의 50-70%
innodb_buffer_pool_instances = 8
```

기본 128MB. 운영에선 반드시 늘림.

### 2.2 sort_buffer / join_buffer / read_rnd_buffer

```ini
sort_buffer_size = 4M
join_buffer_size = 2M
read_rnd_buffer_size = 2M
```

**connection 당 할당** — 너무 크면 메모리 폭증.

### 2.3 tmp_table_size / max_heap_table_size

```ini
tmp_table_size = 64M
max_heap_table_size = 64M
```

GROUP BY / DISTINCT 의 임시 테이블 (메모리). 부족하면 디스크 (느림).

### 2.4 thread_cache_size

```ini
thread_cache_size = 50
```

스레드 재사용. 짧은 connection 많을 때 효과.

---

## 3. I/O

### 3.1 InnoDB I/O

```ini
innodb_io_capacity = 2000        # SSD
innodb_io_capacity_max = 4000
innodb_flush_neighbors = 0       # SSD = 0, HDD = 1
innodb_flush_method = O_DIRECT
```

### 3.2 Redo Log

```ini
innodb_redo_log_capacity = 4G    # 8.0.30+
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 1   # 1 = ACID, 2 = 빠름
```

### 3.3 Doublewrite

```ini
innodb_doublewrite = ON           # crash 안전 — 끄지 말 것
```

ZFS / btrfs 의 atomic write 만 끄기 검토.

---

## 4. Slow Query Log

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1.0
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
log_slow_replica_statements = 1
```

### 4.1 분석

```bash
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log

# 더 강력
pt-query-digest /var/log/mysql/slow.log
```

→ Top 쿼리 = 튜닝 1순위.

---

## 5. Performance Schema

```sql
-- Top 쿼리
SELECT
  digest_text,
  count_star,
  avg_timer_wait/1e9 AS avg_ms,
  sum_rows_examined / count_star AS avg_examined,
  sum_rows_sent     / count_star AS avg_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;

-- 비효율 (examined >> sent)
SELECT digest_text,
  sum_rows_examined / NULLIF(sum_rows_sent, 0) AS ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE sum_rows_sent > 0
ORDER BY ratio DESC LIMIT 20;
```

---

## 6. sys 스키마

```sql
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile LIMIT 10;
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;
SELECT * FROM sys.innodb_lock_waits;
```

---

## 7. 인덱스 튜닝

자세히 → [[indexes]]

### 7.1 빠진 인덱스 찾기

- `EXPLAIN` 의 `type=ALL` + 큰 `rows`
- `Slow Log` + `log_queries_not_using_indexes`
- `sys.statements_with_full_table_scans`

### 7.2 안 쓰이는 인덱스

```sql
SELECT * FROM sys.schema_unused_indexes;
```

→ INVISIBLE → DROP.

---

## 8. 옵티마이저

### 8.1 통계

```sql
ANALYZE TABLE users;
```

옵티마이저가 잘못된 인덱스 → 통계 갱신.

```ini
innodb_stats_persistent = ON
innodb_stats_persistent_sample_pages = 20
```

샘플 페이지 늘리면 정확 ↑, ANALYZE 느림.

### 8.2 옵티마이저 스위치

```sql
SHOW VARIABLES LIKE 'optimizer_switch';

SET optimizer_switch = 'index_merge=on,index_merge_intersection=off';
```

대부분 기본 OK. 특수 케이스에만.

### 8.3 힌트

```sql
SELECT /*+ INDEX(t idx_name) */ ...
SELECT /*+ NO_INDEX(t idx_old) */ ...
SELECT /*+ MAX_EXECUTION_TIME(1000) */ ...
SELECT /*+ JOIN_ORDER(a,b,c) */ ...
SELECT /*+ SEMIJOIN(MATERIALIZATION) */ ...
SELECT /*+ NO_BNL(t) */ ...
```

→ 통계 / 인덱스 정비가 우선. 힌트는 마지막 수단.

---

## 9. Hash Join (8.0.18+)

동등 조건 JOIN 의 가속.

```sql
EXPLAIN FORMAT=TREE SELECT ...;
-- "Hash join" 노드 보임
```

자동 사용. `BNL` 강제는 옛 힌트.

---

## 10. Connection / Pool

```ini
max_connections = 200
thread_cache_size = 50
wait_timeout = 600
```

1 connection ≈ 256 KB. 1000+ 직결은 비효율.

### 10.1 ProxySQL

```
[mysql_servers]:
  rw_host: 1
  ro_host_1, ro_host_2: 2

[mysql_query_rules]:
  ^SELECT → ro_host
  others  → rw_host
```

- 풀링, read/write 분리, 쿼리 라우팅 / 캐싱
- MySQL 표준 풀러.

### 10.2 HikariCP / dbcp2
Java 측 풀. ProxySQL 과 함께 자주 씀.

---

## 11. 쿼리 패턴 튜닝

### 11.1 LIMIT 페이지네이션

```sql
-- ❌ OFFSET 큰 값 = 느림
SELECT * FROM posts ORDER BY id LIMIT 100000, 20;

-- ✅ keyset (cursor)
SELECT * FROM posts WHERE id > :last_id ORDER BY id LIMIT 20;
```

### 11.2 N+1 회피
ORM 의 lazy loading. eager / JOIN 으로.

### 11.3 SELECT *
필요한 컬럼만 — Covering 인덱스 가능성 ↑.

### 11.4 IN ( ) 큰 리스트
1000+ IN — IN 절 너무 큼. 임시 테이블 / JOIN 으로.

### 11.5 BETWEEN AND
인덱스 활용 잘 됨.

### 11.6 OR vs IN
같은 컬럼은 IN, 다른 컬럼은 UNION 으로.

---

## 12. DDL 튜닝

### 12.1 Online DDL

```sql
ALTER TABLE users ADD COLUMN x INT, ALGORITHM=INSTANT;
ALTER TABLE users ADD INDEX idx, ALGORITHM=INPLACE, LOCK=NONE;
```

| ALGORITHM | 의미 |
| --- | --- |
| INSTANT | 즉시 (메타데이터만) — 컬럼 추가 일부 |
| INPLACE | 테이블 복사 X |
| COPY | 전체 복사 |

### 12.2 pt-online-schema-change

```bash
pt-online-schema-change \
  --alter "ADD COLUMN x INT" \
  --execute D=app,t=users
```

큰 테이블의 ALTER 를 lock 없이 (트리거 기반 새 테이블).

### 12.3 gh-ost
GitHub 의 binlog 기반 schema migration. 트리거 X.

---

## 13. 모니터링

### 13.1 핵심 지표

| 지표 | 좋은 값 |
| --- | --- |
| Buffer pool hit rate | > 99% |
| Slow queries | 적음 |
| Replication lag | < 1s |
| Connection 사용률 | < 80% |
| Threads_running | < 코어 수 |
| InnoDB History List Length | 작음 |

### 13.2 도구
- **Percona Monitoring & Management (PMM)** — 추천
- **PlanetScale Insights** (PlanetScale)
- **DataDog / NewRelic** 등 APM
- **Grafana + Prometheus + mysqld_exporter**

---

## 14. OS 튜닝

```bash
# swap 최소화
echo 1 > /proc/sys/vm/swappiness

# THP (Transparent Huge Pages) 끔
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 파일 디스크립터
ulimit -n 65535
```

```ini
# my.cnf
open_files_limit = 65535
```

### 14.1 디스크
- **NVMe SSD** 권장
- ext4 / xfs + `noatime`
- ZFS — 가능하지만 `innodb_doublewrite=OFF` + ZFS atomic

---

## 15. 함정

### 함정 1 — `innodb_buffer_pool_size` 기본 128MB
가장 흔한 미설정. RAM 50-70%.

### 함정 2 — `sort_buffer_size` 너무 큼
connection 당 할당. 폭증.

### 함정 3 — `query_cache` (5.x)
경합 ↑. 비활성 (8.0 에선 아예 제거).

### 함정 4 — Slow Log 안 봄
가장 큰 정보 소스. pt-query-digest 정기.

### 함정 5 — `ANALYZE` 안 함
대량 INSERT 후 통계 부재 → 옵티마이저 오판.

### 함정 6 — DDL 의 long lock
큰 테이블 ALTER 가 분~시간 lock. INSTANT / INPLACE / pt-osc.

### 함정 7 — 인덱스 너무 많음
INSERT/UPDATE 비용 ↑. INVISIBLE 후 검증.

### 함정 8 — `read_only` 누락
실수로 replica 에 쓰기 → split-brain.

### 함정 9 — Adaptive Hash Index 경합
일부 워크로드 off 가 더 빠름.

### 함정 10 — Connection 직결
1000+ 직결 = 메모리 폭발. ProxySQL.

---

## 16. 학습 자료

- **High Performance MySQL** (4th) — 필독
- **Percona Toolkit / PMM**
- **MySQL Reference Manual** Ch. 10 (Optimization)
- **dev.mysql.com/blog** / **Percona Blog**

---

## 17. 관련

- [[configuration]] — 파라미터
- [[indexes]] — 인덱스
- [[explain]] — 쿼리 진단
- [[transactions]] — 락
- [[innodb-engine]] — Buffer Pool 깊이
- [[mysql]] — MySQL hub
