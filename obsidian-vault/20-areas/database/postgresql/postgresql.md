---
title: "PostgreSQL (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:00:00+09:00
tags:
  - database
  - rdb
  - postgresql
  - hub
---

# PostgreSQL (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 12 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**오픈소스 객체-관계형 (ORDB) 데이터베이스**.
1986 UC Berkeley POSTGRES 프로젝트 → 1996 PostgreSQL.
표준 SQL + 확장성 + MVCC + 풍부한 데이터 타입 = "엔터프라이즈급 오픈소스 RDB" 의 사실상 표준.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1986 | Berkeley POSTGRES (Michael Stonebraker) |
| 1995 | Postgres95 (SQL 지원) |
| 1996 | **PostgreSQL 6.0** — 오픈소스 + 이름 변경 |
| 2005 | 8.0 — Windows 지원, PITR |
| 2010 | 9.0 — Streaming Replication, Hot Standby |
| 2016 | 9.6 — 병렬 쿼리 |
| 2017 | 10 — 논리적 복제, 선언적 파티셔닝 |
| 2019 | 12 — Generated Columns, JIT |
| 2022 | 15 — MERGE, 논리적 복제 개선 |
| 2024 | 17 — 점진적 백업, JSON_TABLE |

---

## 3. PostgreSQL 의 특징 (요약)

1. **객체-관계형** — 상속, 사용자 정의 타입
2. **MVCC** — 읽기-쓰기 lock-free
3. **표준 SQL 준수** — ANSI SQL 가장 충실
4. **확장성** — Extension 메커니즘 (PostGIS, pgvector, TimescaleDB)
5. **풍부한 타입** — JSONB, Array, hstore, range, geometric
6. **인덱스 다양성** — B-Tree, Hash, GIN, GiST, BRIN, SP-GiST
7. **트랜잭션 DDL** — `ALTER TABLE` 도 트랜잭션 안에서 (대부분)
8. **WAL 기반** — Durability + Replication + PITR

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / 초기화 / 첫 연결 |
| [[configuration]] | postgresql.conf / pg_hba.conf / 핵심 파라미터 |
| [[data-types]] | 숫자 / 문자 / 날짜 / JSONB / Array / 사용자 정의 |
| [[sql-syntax]] | DDL / DML / DCL / TCL / 윈도우 함수 / CTE |
| [[indexes]] | B-Tree / Hash / GIN / GiST / BRIN / Partial / Expression |
| [[transactions-mvcc]] | ACID / Isolation Level / MVCC / Vacuum |
| [[explain-analyze]] | 실행 계획 읽기 / Cost 모델 / Buffers |
| [[replication]] | Streaming / Logical / Synchronous / Hot Standby |
| [[extensions]] | PostGIS / pgvector / pg_stat_statements / TimescaleDB |
| [[backup-recovery]] | pg_dump / pg_basebackup / PITR / WAL Archiving |
| [[performance-tuning]] | shared_buffers / work_mem / autovacuum / 통계 |
| [[security]] | 패스워드 / 분실 복구 / pg_hba / GRANT 실수 / 외부 노출 |
| [[use-cases]] | 언제 PostgreSQL 을 선택해야 하나 |

---

## 5. PostgreSQL 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 일반 OLTP 웹 서비스 | ✅ 표준 |
| 복잡한 도메인 / JOIN 많음 | ✅ |
| JSON 중심 + 일부 관계형 | ✅ (JSONB) |
| 지리정보 / 위치 | ✅ (PostGIS) |
| 벡터 검색 / AI | ✅ (pgvector) |
| 시계열 | ✅ (TimescaleDB) |
| OLAP / 분석 | ⚠️ — Citus / ClickHouse 검토 |
| 거대 쓰기 (수십만 TPS) | ⚠️ — Cassandra / Scylla |
| 임베디드 / 모바일 | ❌ — SQLite |
| 풀텍스트 검색 (대규모) | ⚠️ — 작으면 OK, 크면 Elasticsearch |

---

## 6. 면접 핵심 질문

1. **MVCC 가 무엇이고 lock 기반과 어떻게 다른가**.
2. **VACUUM 의 역할** — 왜 필요한가, autovacuum 의 동작.
3. **B-Tree vs GIN vs GiST** 의 차이.
4. **EXPLAIN vs EXPLAIN ANALYZE** + Cost 계산.
5. **Isolation Level** — Read Committed / Repeatable Read / Serializable + Phantom read.
6. **Streaming Replication vs Logical Replication**.
7. **PITR (Point-In-Time Recovery)** 의 동작.
8. **JSONB vs JSON**.
9. **shared_buffers, work_mem, effective_cache_size** 튜닝.
10. **Connection Pooling** 이 왜 필요한가 (PgBouncer).

---

## 7. 학습 자료

- **PostgreSQL Documentation** — postgresql.org/docs (가장 좋은 docs)
- **PostgreSQL: Up and Running** (3rd) — Obe / Hsu
- **The Art of PostgreSQL** — Dimitri Fontaine
- **Mastering PostgreSQL 15** — Hans-Jürgen Schönig
- **Postgres Weekly** — newsletter (postgresweekly.com)
- **Crunchy Data Blog** — crunchydata.com/blog

---

## 8. 관련

- [[../mysql/mysql]] — MySQL 비교
- [[../sqlite/sqlite]] — SQLite (PostgreSQL 의 가벼운 사촌)
- [[../database-design/database-design]] — DB 설계
- [[../database-tuning/database-tuning]] — DB 튜닝
- [[../../computer-science/database-theory/database-theory]] — 이론
- [[../database|↑ database hub]]
