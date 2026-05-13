---
title: "PostgreSQL EXPLAIN ANALYZE — 실행 계획 읽기"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:35:00+09:00
tags:
  - database
  - postgresql
  - performance
  - explain
---

# PostgreSQL EXPLAIN ANALYZE

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | EXPLAIN / ANALYZE / Cost / Buffers / 노드 |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. EXPLAIN vs EXPLAIN ANALYZE

```sql
EXPLAIN SELECT ...;             -- 실행 X. 옵티마이저 추정만
EXPLAIN ANALYZE SELECT ...;     -- 실제 실행. 시간 측정
EXPLAIN (ANALYZE, BUFFERS) ...; -- + I/O 정보 (필수)
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON) ...;
```

⚠️ ANALYZE 는 **실제 실행** — UPDATE / DELETE 에 쓰면 데이터 변함. 안전한 패턴:

```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM ...;
ROLLBACK;
```

---

## 2. 기본 출력 읽기

```
Seq Scan on users  (cost=0.00..18334.00 rows=10000 width=100)
                   (actual time=0.012..120.345 rows=9876 loops=1)
```

| 필드 | 의미 |
| --- | --- |
| `cost=A..B` | A = 시작 비용, B = 전체 추정 비용 (page 단위 가상 비용) |
| `rows=N` | 옵티마이저의 추정 행 수 |
| `width=N` | 행당 추정 byte |
| `actual time=A..B` | 첫 행까지 ms .. 마지막 행까지 ms |
| `rows=N` (actual) | 실제 행 수 |
| `loops=N` | 이 노드가 실행된 횟수 |

### 2.1 핵심 — Estimate vs Actual

```
rows=10000   ← 추정
rows=9876    ← 실제
```
큰 차이가 나면 **통계 갱신 필요** (`ANALYZE table`) 또는 인덱스 / 쿼리 재검토.

---

## 3. 주요 노드 (스캔)

### 3.1 Seq Scan
테이블 전체 읽기. 작은 테이블 / 큰 비율 결과에는 OK. 큰 테이블에서 일부만 찾는데 Seq → 인덱스 부재 의심.

### 3.2 Index Scan
인덱스로 위치 찾고 테이블로 가서 행 읽음.

### 3.3 Index Only Scan
인덱스만 읽고 끝 (Covering Index). 가장 빠름.

```
Index Only Scan using idx_users_email on users
  Heap Fetches: 0   ← 0 이면 완전한 Index Only Scan
```

### 3.4 Bitmap Index Scan + Bitmap Heap Scan
여러 행 / 여러 인덱스 결합 시. 인덱스로 비트맵 만들고 한 번에 테이블 페치.

### 3.5 Tid Scan
`WHERE ctid = '...'` — 거의 안 씀.

---

## 4. 주요 노드 (JOIN)

### 4.1 Nested Loop
외부 테이블 행마다 내부 검색. **외부 작음 + 내부 인덱스** 일 때 최고.

```
Nested Loop  (cost=...)
  -> Seq Scan on small (rows=10)
  -> Index Scan on big (rows=1 per loop)
```

### 4.2 Hash Join
한 쪽으로 hash 테이블 만들고 다른 쪽 probe. **중간 크기** 양쪽일 때 좋음.

```
Hash Join
  -> Seq Scan on a
  -> Hash
    -> Seq Scan on b   ← 메모리 가능한 작은 쪽
```

### 4.3 Merge Join
양쪽 정렬 후 병합. **둘 다 정렬되어 있거나 인덱스** 일 때.

### 4.4 어떤 JOIN 이 빠른가
- **작은 + 인덱스** → Nested Loop
- **중간 + 중간** → Hash
- **거대 + 거대 + 정렬** → Merge

---

## 5. 정렬 / 집계

### 5.1 Sort

```
Sort
  Sort Key: created_at
  Sort Method: quicksort  Memory: 25kB
  -> ...

또는
  Sort Method: external merge  Disk: 1024kB    ← 디스크로 떨어짐 (느림)
```

`work_mem` 부족 → 디스크. 늘리거나 인덱스로 정렬 회피.

### 5.2 Aggregate / HashAggregate

```
HashAggregate
  Group Key: role
  -> Seq Scan on users
```

`work_mem` 부족 → `Aggregate` (정렬 기반).

### 5.3 GroupAggregate
입력이 이미 정렬되어 있을 때.

---

## 6. BUFFERS — I/O 진단

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

```
Buffers: shared hit=120 read=45 dirtied=10 written=5
```

| 항목 | 의미 |
| --- | --- |
| `hit` | shared_buffers 에서 적중 (RAM) |
| `read` | OS / 디스크에서 읽음 |
| `dirtied` | 더럽혀진 페이지 (이 쿼리에서 쓰기) |
| `written` | 디스크에 쓴 페이지 |
| `temp written/read` | 임시 파일 (work_mem 부족 시) |

**`read` 가 크면 I/O 가 병목** — 인덱스 / 캐시 / shared_buffers 검토.

---

## 7. 병렬 (Parallel)

```
Gather  (cost=... rows=10000)
  Workers Planned: 2
  Workers Launched: 2
  -> Parallel Seq Scan on big_table
```

`max_parallel_workers_per_gather`, `parallel_setup_cost`, `parallel_tuple_cost` 로 조정.

---

## 8. CTE / 서브쿼리

- PG 12+ 에서 CTE 는 inline 가능 (`MATERIALIZED` / `NOT MATERIALIZED`)
- `WITH x AS (SELECT ...)` → optimizer fence 가 약해짐

```sql
WITH active AS NOT MATERIALIZED (SELECT * FROM users WHERE active)
SELECT * FROM active WHERE role = 'admin';
-- inline 되어 함께 최적화
```

---

## 9. 실전 분석 예

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, COUNT(p.id)
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
WHERE u.created_at > now() - INTERVAL '30 days'
GROUP BY u.id;
```

체크포인트:
1. `actual rows` vs `rows` — 추정 정확한가?
2. Seq Scan 인가, Index Scan 인가?
3. JOIN 방식이 적절한가?
4. Sort 가 메모리에서 끝나는가?
5. `Buffers: read` 가 큰가?
6. `actual time` 의 비중이 어디에 가장 큰가?

---

## 10. auto_explain

느린 쿼리만 자동으로 실행 계획 로그.

```ini
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 500ms
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_format = json
```

---

## 11. pg_stat_statements

쿼리별 통계 — 전체 시스템 탑 N 쿼리.

```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## 12. 시각화

- **explain.depesz.com** — 가장 유명. JSON 붙여 넣기.
- **explain.dalibo.com** — 트리 시각화.
- **pgMustard** — 자동 권장.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
-- 결과 JSON 을 위 사이트에 붙여넣기
```

---

## 13. 함정

### 함정 1 — EXPLAIN ANALYZE 의 부작용
실제 실행. UPDATE/DELETE 는 BEGIN/ROLLBACK 으로 감싸기.

### 함정 2 — Cost 와 실제 시간 다름
Cost 는 추정. ANALYZE 의 actual time 이 진실.

### 함정 3 — actual rows × loops 가 진짜
Nested Loop 안쪽의 `rows=1 loops=1000` → 실제 1000 행.

### 함정 4 — 캐시 워밍 차이
첫 실행은 cold, 두 번째는 warm. 비교 시 `pg_prewarm` 또는 여러 번 실행.

### 함정 5 — 통계 오래됨
`pg_stat_user_tables.last_analyze` 확인. `ANALYZE table` 로 갱신.

### 함정 6 — `Heap Fetches > 0` 의 Index Only Scan
Visibility map 미준비 → VACUUM 으로 갱신.

---

## 14. 학습 자료

- PostgreSQL Documentation Ch. 14 (Performance Tips)
- **explain.depesz.com**
- **PostgreSQL 14 Internals** — Rogov

---

## 15. 관련

- [[indexes]] — 인덱스 선정
- [[performance-tuning]] — 튜닝
- [[postgresql]] — PostgreSQL hub
