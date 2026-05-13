---
title: "MySQL EXPLAIN — 실행 계획 / 옵티마이저 트레이스"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:50:00+09:00
tags:
  - database
  - mysql
  - explain
  - performance
---

# MySQL EXPLAIN

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | EXPLAIN / FORMAT JSON / ANALYZE |

**[[mysql|↑ MySQL hub]]**

---

## 1. EXPLAIN 기본

```sql
EXPLAIN SELECT ...;
EXPLAIN FORMAT=JSON SELECT ...\G
EXPLAIN ANALYZE SELECT ...;        -- 8.0.18+ (실제 실행)
EXPLAIN FOR CONNECTION 123;        -- 다른 세션의 실행 계획
```

---

## 2. EXPLAIN 출력 — 열

```
id  select_type  table   partitions  type   possible_keys  key   key_len  ref  rows  filtered  Extra
```

| 열 | 의미 |
| --- | --- |
| `id` | SELECT 식별자 (같으면 JOIN, 다르면 서브쿼리) |
| `select_type` | SIMPLE / PRIMARY / SUBQUERY / DERIVED / UNION |
| `table` | 테이블 (또는 `<derived2>` 등) |
| `type` | 접근 방식 (아래 3 절) |
| `possible_keys` | 옵티마이저가 고려한 인덱스 |
| `key` | 실제 사용된 인덱스 |
| `key_len` | 사용된 인덱스 byte |
| `ref` | 인덱스의 비교 대상 (상수, 컬럼) |
| `rows` | 추정 검사 행 수 |
| `filtered` | 필터 후 남는 % |
| `Extra` | 추가 정보 (가장 중요) |

---

## 3. type — 접근 방식 (좋은 순서)

| type | 의미 | 좋음 / 나쁨 |
| --- | --- | --- |
| `system` | 1 행 시스템 테이블 | ✨ 최고 |
| `const` | PK / UNIQUE 1 행 | ✨ 최고 |
| `eq_ref` | JOIN 시 PK/UNIQUE 1 행 | ✨ 최고 |
| `ref` | 인덱스 prefix 사용 | ✅ 좋음 |
| `range` | 범위 (`<, BETWEEN`) | ✅ 좋음 |
| `index` | 인덱스 전체 스캔 | ⚠️ |
| `ALL` | 테이블 전체 스캔 | ❌ 큰 테이블에 위험 |

목표: **ref / range 이상**. 큰 테이블의 `ALL` 은 적신호.

---

## 4. Extra — 핵심 신호

| Extra | 의미 |
| --- | --- |
| `Using index` | ✨ Index Only Scan (Covering) |
| `Using where` | WHERE 필터 적용 |
| `Using filesort` | ⚠️ 정렬 — 인덱스 활용 못 함 |
| `Using temporary` | ⚠️ 임시 테이블 — GROUP BY / DISTINCT |
| `Using index condition` | ICP (Index Condition Pushdown) |
| `Using join buffer` | 인덱스 없는 JOIN — block nested loop |
| `Using MRR` | Multi-Range Read |
| `Backward index scan` | 역순 (8.0+) |
| `Range checked for each record` | 매 행 검사 — 매우 나쁨 |

### 4.1 `Using filesort` 가 보이면
ORDER BY 가 인덱스로 충족 안 됨. 인덱스에 정렬 컬럼 포함 / 순서 조정.

### 4.2 `Using temporary` 가 보이면
GROUP BY 가 인덱스로 충족 안 됨. 인덱스 / 쿼리 재작성.

---

## 5. EXPLAIN FORMAT=JSON

```sql
EXPLAIN FORMAT=JSON SELECT ...\G
```

추가 정보:
- `cost_info` — 추정 비용
- `attached_condition` — 적용된 조건
- `materialized_from_subquery`
- `nested_loop`, `attached_subqueries`

→ MySQL Workbench 의 Visual Explain 이 이걸 시각화.

---

## 6. EXPLAIN ANALYZE (8.0.18+)

**실제 실행** + 시간 측정.

```sql
EXPLAIN ANALYZE SELECT u.name, COUNT(p.id)
FROM users u LEFT JOIN posts p ON p.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL 30 DAY
GROUP BY u.id\G
```

```
-> Aggregate ...  (actual time=2.345..2.567 rows=10 loops=1)
   -> Left hash join ...  (actual time=1.234..2.123 rows=120 loops=1)
      -> Filter ...  (actual time=0.012..0.234 rows=10 loops=1)
      -> Hash
         -> Table scan on p  (actual time=0.034..0.567 rows=1000)
```

| 항목 | 의미 |
| --- | --- |
| `actual time=A..B` | 첫 행 ms .. 마지막 행 ms |
| `rows=N loops=M` | 실제 행 × 실행 횟수 |

⚠️ ANALYZE 는 **실제 실행** — UPDATE/DELETE 는 BEGIN/ROLLBACK.

---

## 7. 옵티마이저 트레이스 (Optimizer Trace)

옵티마이저가 어떻게 결정했는지.

```sql
SET optimizer_trace="enabled=on";
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace="enabled=off";
```

비용 추정 / 대안 비교가 보임. 깊은 분석.

---

## 8. Slow Query Log + Index Usage

```ini
slow_query_log = 1
long_query_time = 1.0
log_queries_not_using_indexes = 1
```

```bash
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log
pt-query-digest /var/log/mysql/slow.log     # Percona Toolkit
```

---

## 9. Performance Schema

```sql
-- 가장 시간 잡아먹는 쿼리
SELECT digest_text, count_star, avg_timer_wait/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;

-- 인덱스 사용 통계
SELECT * FROM sys.schema_index_statistics ORDER BY rows_selected DESC LIMIT 20;
SELECT * FROM sys.schema_unused_indexes;
```

---

## 10. 시각화

- **MySQL Workbench Visual Explain** — Workbench 내장
- **explain.depesz.com** — PostgreSQL 용 (MySQL 미지원)
- **Percona Toolkit `pt-visual-explain`** — 텍스트 트리

---

## 11. 자주 보는 실행 계획 예

### 11.1 PK 조회

```
id  type    key      rows  Extra
1   const   PRIMARY  1
```
이상적.

### 11.2 인덱스 + 정렬

```
id  type   key                       Extra
1   range  idx_user_created          Using index condition
```

### 11.3 ORDER BY 가 인덱스로 안 풀림

```
id  type  Extra
1   ALL   Using where; Using filesort
```
→ ORDER BY 컬럼 인덱스 검토.

### 11.4 JOIN

```
id  table  type   key
1   u      ref    idx_email
1   p      ref    idx_user_id
```
양쪽 ref 면 보통 OK.

---

## 12. JOIN 알고리즘

| 알고리즘 | 의미 |
| --- | --- |
| **Block Nested Loop (BNL)** | 인덱스 없는 JOIN — block 단위 |
| **Nested Loop Join** | 인덱스 있는 JOIN |
| **Hash Join** (8.0.18+) | 동등 조건 + 메모리 |

```sql
-- 강제 옵티마이저 힌트
SELECT /*+ USE_INDEX (orders idx_user_id) */ ...
SELECT /*+ JOIN_ORDER (a, b, c) */ ...
SELECT STRAIGHT_JOIN ...
```

힌트 남발 X — 통계 / 인덱스 정비가 먼저.

---

## 13. 자주 쓰는 옵티마이저 힌트 (8.0+)

```sql
SELECT /*+ INDEX(t idx_name) */ ...
SELECT /*+ NO_INDEX(t idx_old) */ ...
SELECT /*+ MAX_EXECUTION_TIME(1000) */ ...
SELECT /*+ SET_VAR(sort_buffer_size=1024*1024) */ ...
SELECT /*+ MERGE() */ ...
SELECT /*+ NO_BNL(t1,t2) */ ...
```

---

## 14. ANALYZE TABLE — 통계 갱신

```sql
ANALYZE TABLE users;
```

옵티마이저가 잘못된 인덱스 선택 → 통계 오래됐을 가능성. `innodb_stats_persistent` 기본 ON.

---

## 15. 함정

### 함정 1 — `rows` 가 추정
실제 행 수는 ANALYZE 가 정확.

### 함정 2 — `EXPLAIN ANALYZE` 의 부작용
실제 실행 — DELETE/UPDATE 는 트랜잭션 안에서.

### 함정 3 — JSON 으로만 보면 놓침
Visual / Text 둘 다 보기.

### 함정 4 — Workbench Visual Explain 의 화살표 방향
실행 순서와 다를 수 있음 — 트리 읽기.

### 함정 5 — 캐시 효과
첫 실행은 cold, 이후 warm. 측정은 여러 번.

### 함정 6 — 통계 오래됨
`ANALYZE TABLE` 후 재측정.

### 함정 7 — `index` vs `Using index`
`type=index` (전체 인덱스 스캔, 나쁨) vs `Extra: Using index` (Covering, 좋음). 다른 의미.

---

## 16. 학습 자료

- **MySQL Reference Manual** Ch. 10 (Optimization)
- **High Performance MySQL** Ch. 8 (Optimizing Queries)
- **Percona Toolkit** — pt-query-digest
- **MySQL Workbench Documentation**

---

## 17. 관련

- [[indexes]] — 인덱스 설계
- [[performance-tuning]] — 튜닝
- [[transactions]] — 락 진단
- [[mysql]] — MySQL hub
