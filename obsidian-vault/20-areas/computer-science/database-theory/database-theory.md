---
title: "데이터베이스 이론 (Database Theory)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T11:00:00+09:00
tags:
  - database
  - sql
  - transaction
  - index
  - normalization
---

# 데이터베이스 이론 (Database Theory)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

> 이 노트는 **이론 / 개념** 중심. 실무 DB 운영 (PostgreSQL/MySQL 튜닝) 은
> [[../../database/database|↑ database (areas)]] 참조.

---

## 1. 한 줄 정의

**구조화된 데이터의 영속 저장 + 동시 접근 + 일관성** 을 책임지는 시스템.
관계 모델 (1970 Codd), 트랜잭션 (ACID), 인덱스, 쿼리 옵티마이저가 핵심.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1970 | Edgar F. Codd — 관계 모델 (RDBMS 의 토대) |
| 1974 | IBM System R — 첫 RDBMS 프로토타입 |
| 1979 | Oracle 출시 (Larry Ellison) |
| 1986 | SQL ANSI 표준 |
| 1995 | MySQL 1.0 |
| 1996 | PostgreSQL 6.0 (Berkeley Postgres 후속) |
| 2003 | Google File System / Bigtable (NoSQL 시작) |
| 2006 | Dynamo Paper (Amazon) — eventual consistency |
| 2009 | NoSQL 운동 (MongoDB, Cassandra, Redis) |
| 2012 | Google Spanner — globally distributed ACID |
| 2014 | NewSQL (CockroachDB, TiDB) |
| 2020+ | Vector DB (pgvector, Pinecone, Weaviate) — RAG |

핵심 통찰: **트랜잭션 (Jim Gray, 1981) + 관계 모델 (Codd 1970) = RDBMS 의 양대 기둥**.

---

## 3. 관계 모델

### 3.1 핵심 개념

- **Relation** (테이블) = 튜플의 집합
- **Tuple** (행) = 속성 값의 집합
- **Attribute** (열) = 도메인의 값
- **Domain** = 가능한 값의 집합 (예: INT)
- **Schema** = 테이블 정의 (속성 + 타입)

### 3.2 키

| 키 | 정의 |
| --- | --- |
| **Super Key** | 행을 유일하게 식별하는 속성 집합 |
| **Candidate Key** | 최소 super key |
| **Primary Key** | 선택된 candidate key |
| **Foreign Key** | 다른 테이블의 PK 참조 |
| **Alternate Key** | 선택되지 않은 candidate key |
| **Composite Key** | 여러 속성으로 구성 |

### 3.3 관계 대수 (Relational Algebra)

| 연산 | 기호 | 의미 |
| --- | --- | --- |
| Selection | σ | WHERE |
| Projection | π | SELECT 컬럼 |
| Union | ∪ | UNION |
| Intersection | ∩ | INTERSECT |
| Difference | − | EXCEPT |
| Cartesian Product | × | CROSS JOIN |
| Join | ⋈ | INNER/OUTER JOIN |
| Rename | ρ | AS |

SQL 의 이론적 기반.

---

## 4. 정규화 (Normalization)

### 4.1 정규형

| 정규형 | 조건 |
| --- | --- |
| **1NF** | 원자 값 (atomic) — 배열/중복 컬럼 없음 |
| **2NF** | 1NF + 부분 함수 종속 제거 (PK 전체에 종속) |
| **3NF** | 2NF + 이행적 함수 종속 제거 (PK → A → B 금지) |
| **BCNF** | 3NF + 모든 비자명 FD 의 결정자가 super key |
| **4NF** | BCNF + 다중값 종속 제거 |
| **5NF** | 4NF + 조인 종속 제거 |

### 4.2 함수 종속 (Functional Dependency)

X → Y: X 값이 같으면 Y 값도 같음.

예) `student_id → name, dept`

### 4.3 비정규화 (Denormalization)

- 성능을 위해 조인 회피
- 데이터 중복 허용
- 갱신 이상 / 삽입 이상 / 삭제 이상 위험
- 분석 / 데이터 웨어하우스 (Star Schema)

### 4.4 정규화 vs 비정규화

| 측면 | 정규화 | 비정규화 |
| --- | --- | --- |
| 무결성 | 높음 | 낮음 |
| 쓰기 | 빠름 | 느림 |
| 읽기 | 느림 (조인) | 빠름 |
| 저장 | 작음 | 큼 |
| 용도 | OLTP | OLAP |

---

## 5. 트랜잭션 — ACID

### 5.1 ACID

- **Atomicity (원자성)** — 모두 성공 / 모두 실패
- **Consistency (일관성)** — 제약 조건 유지
- **Isolation (격리성)** — 동시 트랜잭션 간 간섭 없음
- **Durability (지속성)** — 커밋된 결과는 영구

### 5.2 격리 수준 (SQL 표준)

| 수준 | Dirty Read | Non-repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE READ | ❌ | ❌ | ✅ |
| SERIALIZABLE | ❌ | ❌ | ❌ |

PostgreSQL 의 REPEATABLE READ 는 사실상 Snapshot Isolation.
MySQL InnoDB 의 REPEATABLE READ 는 next-key locking 으로 phantom 도 막음.

### 5.3 동시성 이상

- **Dirty Read** — 커밋 안 된 데이터 읽기
- **Non-repeatable Read** — 같은 SELECT 가 다른 결과
- **Phantom Read** — 같은 WHERE 가 다른 행 수
- **Write Skew** — Snapshot Isolation 에서 가능
- **Lost Update** — 두 트랜잭션이 같은 행 덮어씀

### 5.4 동시성 제어

#### (a) Lock-based (2PL — Two-Phase Locking)
- **Growing Phase** — 락 획득만
- **Shrinking Phase** — 락 해제만
- 직렬화 가능성 보장. 데드락 위험.

#### (b) MVCC (Multi-Version Concurrency Control)
- 각 버전에 트랜잭션 ID 부착
- 읽기는 락 없음 — 자기 시점의 버전 읽기
- PostgreSQL, MySQL (InnoDB), Oracle 의 기본
- VACUUM (PostgreSQL) — 오래된 버전 정리

#### (c) OCC (Optimistic Concurrency Control)
- 충돌 가정 안 함, 커밋 시 검증
- Spanner, CockroachDB 의 일부

---

## 6. 인덱스

### 6.1 인덱스 자료구조

| 자료구조 | 특징 | 사용 |
| --- | --- | --- |
| **B+Tree** | 균형 / 디스크 친화 / 범위 질의 | OLTP 기본 |
| **B-Tree** | leaf 에 데이터 | NTFS, 일부 DB |
| **Hash** | 동등 비교만 | Memcached, 메모리 인덱스 |
| **LSM-Tree** | 쓰기 최적화 | Cassandra, RocksDB |
| **R-Tree** | 공간 인덱스 | PostGIS |
| **GIN / GiST** | 역색인 / 일반화 | PostgreSQL 전문 검색 |
| **Bitmap** | 카디널리티 낮은 컬럼 | DW |
| **HNSW / IVF** | 벡터 검색 | Pinecone, pgvector |

### 6.2 B+Tree 가 표준인 이유

- 디스크 페이지 크기 (4KB / 8KB) 와 노드 정렬
- 한 노드에 100-1000 키 → 트리 높이 3-4 로 10 억 행 처리
- 범위 질의 — leaf 연결 리스트
- 균형 유지 (split / merge)

### 6.3 인덱스 종류

| 종류 | 설명 |
| --- | --- |
| **Primary Index** | PK 기반 |
| **Secondary Index** | 비-PK 기반 |
| **Composite Index** | 여러 컬럼 — 순서 중요 |
| **Unique Index** | 유일성 강제 |
| **Partial Index** | WHERE 조건 부분만 |
| **Covering Index** | SELECT 컬럼 모두 인덱스에 포함 |
| **Functional Index** | `LOWER(email)` 등 |

### 6.4 인덱스 트레이드오프

- 읽기 빠름 ↔ 쓰기 느림 (인덱스 갱신)
- 디스크 공간 증가
- 카디널리티 낮은 컬럼은 비효율

### 6.5 인덱스 안 타는 경우

- `WHERE LOWER(name) = 'kim'` — 함수 적용 → functional index
- `WHERE name LIKE '%kim%'` — 앞 와일드카드
- `WHERE age + 1 = 30` — 좌변 연산
- 데이터의 30%+ 반환 — 풀스캔이 빠를 수 있음
- 통계 부정확 — `ANALYZE` 필요

---

## 7. 쿼리 옵티마이저

### 7.1 동작

```
SQL → Parser → AST → Logical Plan → Optimizer → Physical Plan → Executor
```

### 7.2 비용 기반 (Cost-Based Optimizer, CBO)

- 통계 (행 수, 컬럼 분포, 인덱스 selectivity)
- 가능한 plan 탐색 — 가장 싼 것 선택
- Bushy / Left-deep / Right-deep join trees

### 7.3 Join 알고리즘

| 알고리즘 | 시간 | 조건 |
| --- | --- | --- |
| **Nested Loop** | O(NM) | 작은 inner / index |
| **Hash Join** | O(N+M) | 등호 join, 큰 메모리 |
| **Sort-Merge Join** | O(N log N + M log M) | 정렬됨 / 범위 |

### 7.4 EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'KR';

-- 출력: Plan, Cost, Actual rows, Time
```

PostgreSQL `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`.

---

## 8. 트랜잭션 로그 / 복구

### 8.1 WAL (Write-Ahead Logging)

- 데이터 변경 전 로그 먼저
- ARIES 알고리즘 (IBM 1992)
- PostgreSQL, MySQL, SQLite 모두 사용

### 8.2 체크포인트

- WAL 의 변경 사항을 디스크 적용
- 복구 시간 단축

### 8.3 복구

- **REDO** — 커밋된 트랜잭션 재적용
- **UNDO** — 미커밋 트랜잭션 되돌리기

### 8.4 백업

- **물리 백업** — 데이터 파일 / WAL (PITR — Point-in-Time Recovery)
- **논리 백업** — `pg_dump`, `mysqldump`

---

## 9. CAP 정리 & PACELC

### 9.1 CAP (Brewer 2000)

분산 DB 에서 동시에 3 개 보장 불가:
- **C** onsistency
- **A** vailability
- **P** artition tolerance

네트워크 분할 (P) 은 현실 → **CP** 또는 **AP** 중 선택.

### 9.2 PACELC (Abadi 2012)

- **P** artition 시 → **A** vailability or **C** onsistency
- **E** lse → **L** atency or **C** onsistency

| DB | CAP | PACELC |
| --- | --- | --- |
| PostgreSQL (단일) | CA | EC |
| MongoDB | CP | EC |
| Cassandra | AP | EL |
| DynamoDB | AP | EL |
| Spanner | CP | EC (TrueTime) |

### 9.3 일관성 수준

- **Strong** — 모든 읽기가 최신
- **Eventual** — 시간이 지나면 일관
- **Read-your-writes** — 자기 쓰기는 보임
- **Monotonic Read** — 시간이 거꾸로 안 감
- **Bounded Staleness** — N 초 / N 버전 지연 허용

---

## 10. NoSQL 분류

| 종류 | 모델 | 예 |
| --- | --- | --- |
| **Key-Value** | dict | Redis, DynamoDB, Memcached |
| **Document** | JSON | MongoDB, CouchDB |
| **Wide-Column** | 행 X 컬럼 패밀리 | Cassandra, HBase, Bigtable |
| **Graph** | 노드 + 간선 | Neo4j, JanusGraph |
| **Time-Series** | 타임스탬프 | InfluxDB, TimescaleDB |
| **Search** | 역색인 | Elasticsearch, OpenSearch |
| **Vector** | 임베딩 | Pinecone, pgvector, Milvus |

---

## 11. 함정 / 안티패턴

### 함정 1 — N+1 쿼리
```python
for user in users:           # 1 쿼리
    print(user.orders)       # N 쿼리
# → JOIN 또는 IN 으로 1 쿼리
```

### 함정 2 — SELECT *
- 네트워크 / 메모리 낭비
- Covering Index 무효화

### 함정 3 — 인덱스 남용
- 쓰기 성능 저하
- 작은 테이블엔 풀스캔이 빠름

### 함정 4 — 트랜잭션 너무 김
- 락 보유 시간 증가 → 동시성 저하
- DB 외 작업 (API 호출) 트랜잭션 안에서 X

### 함정 5 — UPDATE 의 WHERE 빠뜨림
모든 행 갱신. 항상 트랜잭션 + LIMIT.

### 함정 6 — 외래 키 없이 참조
무결성 깨짐. ORM 이 자동 ON DELETE CASCADE 하면 위험.

### 함정 7 — 격리 수준 default 가정
PostgreSQL READ COMMITTED, MySQL InnoDB REPEATABLE READ. 명시.

### 함정 8 — SQL Injection
파라미터 바인딩 필수. 문자열 concat 절대 X.

---

## 12. 분산 / Sharding

### 12.1 Replication

- **Master-Slave** — 읽기 분산 / 쓰기는 master
- **Multi-Master** — 충돌 해결 필요
- **Synchronous** vs **Asynchronous**

### 12.2 Sharding

- 데이터를 키로 분할
- **Range** — 알파벳, 시간
- **Hash** — `hash(key) % N`
- **Consistent Hashing** — 노드 추가 시 재분배 최소
- **Directory** — 메타 매핑 테이블

### 12.3 분산 트랜잭션

- **2PC (Two-Phase Commit)** — Prepare → Commit, blocking
- **3PC** — non-blocking 시도, 잘 안 씀
- **Saga** — 보상 트랜잭션 체인
- **TCC** (Try-Confirm-Cancel)

---

## 13. 면접 질문

1. **ACID 와 격리 수준**.
2. **B+Tree 가 표준인 이유**.
3. **MVCC 가 무엇이고 어떻게 동작하나**.
4. **인덱스가 안 타는 경우**.
5. **정규형 vs 비정규화** — 어느 때 무엇.
6. **N+1 문제**.
7. **CAP / PACELC**.
8. **NoSQL vs RDBMS**.
9. **트랜잭션 격리 수준의 이상 현상**.
10. **분산 트랜잭션 (Saga / 2PC)**.

---

## 14. 학습 자료

- **Database System Concepts** (Silberschatz / Korth / Sudarshan) — 표준 교재
- **Designing Data-Intensive Applications** (Martin Kleppmann) — 분산 DB 바이블
- **Readings in Database Systems** (Red Book, Stonebraker) — 논문 모음
- **PostgreSQL Documentation** — 가장 잘 쓴 매뉴얼
- **CMU 15-445 / 15-721** — Andy Pavlo 강의 (YouTube)
- Codd 1970 원 논문 — "A Relational Model of Data for Large Shared Data Banks"

---

## 15. 관련

- [[../../database/database|↑ database (실무)]]
- [[../distributed-systems/distributed-systems]] — CAP, Spanner
- [[../data-structure/trees/trees]] — B-Tree, LSM-Tree
- [[../data-structure/hash-tables/hash-tables]] — Hash Index
- [[../computer-science|↑ computer-science]]
