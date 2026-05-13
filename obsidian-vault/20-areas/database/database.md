---
title: "데이터베이스 (Database) — Hub"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - index
  - database
  - hub
---

# 데이터베이스 (Database) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 hub 재편 — 각 DB 별 폴더로 깊이 분리 (설정 / 문법 / 개념 / 사용처) |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 폴더링 골격 |

**[[../areas|↑ 20-areas]]**

> 이 노트는 **데이터베이스 전체의 hub**. 각 DB 는 별도 폴더 / 노트로 깊이 다룬다.
> 이론은 `computer-science/database-theory`, backend 통합은 `backend/<framework>/db-connection.md`.

---

## 0. 데이터베이스가 다루는 영역

데이터베이스는 **데이터의 저장 / 조회 / 일관성 / 영속성** 을 다룬다:
- 모델 — Relational (관계), Document, Key-Value, Wide-Column, Graph, Search
- 트랜잭션 — ACID / BASE
- 인덱스 — B-Tree, Hash, GIN, GiST, LSM-Tree
- 일관성 — Strong / Eventual / Tunable
- 분산 — Replication, Sharding, Partition Tolerance

System R 1974, PostgreSQL 1986, MySQL 1995, MongoDB 2009, Redis 2009 까지 누적이 오늘의 DB 생태.

---

## 1. RDB (관계형) — 깊이

| DB | 노트 | 한 줄 |
| --- | --- | --- |
| **PostgreSQL** | [[postgresql/postgresql\|↗ PostgreSQL hub]] | 표준 SQL + MVCC + 확장성. OLTP / OLAP 모두 |
| **MySQL** | [[mysql/mysql\|↗ MySQL hub]] | InnoDB + 단순성 + 거대한 생태계. 웹 표준 |
| **SQLite** | [[sqlite/sqlite\|↗ SQLite hub]] | 단일 파일 임베디드. 로컬 / 모바일 / 테스트 |

---

## 2. NoSQL — 깊이

| DB | 분류 | 노트 |
| --- | --- | --- |
| **Redis** | Key-Value (in-memory) | [[redis/redis\|↗ Redis hub]] |
| **MongoDB** | Document | [[mongodb/mongodb\|↗ MongoDB hub]] |
| **Cassandra** | Wide-Column | [[cassandra/cassandra\|↗ Cassandra hub]] |

---

## 3. 검색 엔진 — 깊이

| DB | 노트 |
| --- | --- |
| **Elasticsearch / OpenSearch** | [[elasticsearch/elasticsearch\|↗ Elasticsearch hub]] |

---

## 4. 횡단 영역 — 설계 / 튜닝

- [[database-design/database-design|↗ DB 설계 실무]] — 정규화 / ERD / 인덱스 전략 / migrations
- [[database-tuning/database-tuning|↗ DB 튜닝]] — 쿼리 최적화 / 통계 / EXPLAIN / 캐시

---

## 4.5. 횡단 영역 — 보안 / 패스워드

DB 별 보안 노트 — 패스워드 설정 / 분실 복구 / 권한 흔한 실수 / 외부 노출 사고:

| DB | 노트 | 핵심 사고 패턴 |
| --- | --- | --- |
| **PostgreSQL** | [[postgresql/security\|↗ PG 보안]] | pg_hba.conf 인증 / SCRAM / `public` schema 권한 |
| **MySQL** | [[mysql/security\|↗ MySQL 보안]] | `'user'@'host'` 매칭 / `'root'@'%'` / `--skip-grant-tables` 복구 |
| **Redis** | [[redis/security\|↗ Redis 보안]] | 인증 없는 외부 노출 → RCE / cryptojacking |
| **MongoDB** | [[mongodb/security\|↗ MongoDB 보안]] | `--auth` 미설정 + `bindIp 0.0.0.0` → 랜섬 |

공통 원칙:
- **평문 패스워드는 "찾을" 수 없다** — 모두 해시 저장. 분실 시 OS 권한으로 재설정.
- **최소 권한** — 앱 유저는 자기 DB / SELECT-INSERT-UPDATE-DELETE 만.
- **bind / firewall / TLS** 삼중 방어. 어느 하나 빠지면 사고.
- **운영 패스워드는 vault** (1Password / Vault / AWS Secrets Manager). `.env` 평문 X.

---

## 5. CAP / 일관성 모델 비교

| 모델 | 일관성 | 가용성 | 분할 내성 | 예 |
| --- | --- | --- | --- | --- |
| CP | ✅ | ⚠️ | ✅ | MongoDB (기본), HBase |
| AP | ⚠️ (eventual) | ✅ | ✅ | Cassandra, DynamoDB |
| CA | ✅ | ✅ | ❌ | 단일 노드 RDB |

분산 환경에서는 P 가 강제 → CP / AP 중 선택.

---

## 6. 어떤 DB 를 언제 쓰나 — 결정 표

| 시나리오 | 추천 | 이유 |
| --- | --- | --- |
| 일반 웹 서비스 / 도메인 모델 명확 | PostgreSQL / MySQL | ACID, JOIN, 일관성 |
| 단순 KV / 캐시 / 세션 / 분산 락 | Redis | μs 응답, in-memory |
| 스키마 유연성 / JSON 중심 / MVP | MongoDB | 문서 모델, 유연 |
| 시계열 / 거대 쓰기 / 분산 | Cassandra | 선형 확장, AP |
| 풀텍스트 검색 / 로그 분석 | Elasticsearch | 역색인 |
| 모바일 / 데스크탑 / 단일 파일 | SQLite | 임베디드 |
| 그래프 (소셜 / 추천) | Neo4j (별도) | 관계 traversal |
| OLAP / 분석 / 거대 데이터 | ClickHouse / BigQuery (별도) | 컬럼 저장 |

---

## 7. 핵심 키워드 한 줄

| 키워드 | 의미 |
| --- | --- |
| ACID | Atomicity / Consistency / Isolation / Durability |
| BASE | Basically Available / Soft state / Eventual |
| MVCC | Multi-Version Concurrency Control |
| B-Tree | 디스크 친화 균형 트리 (RDB 인덱스 표준) |
| LSM-Tree | 쓰기 최적 (Cassandra, RocksDB) |
| WAL | Write-Ahead Log (Durability 의 핵심) |
| Replication | 동기 / 비동기 복제 |
| Sharding | 데이터 수평 분할 |
| Index | 조회 가속 자료구조 |
| Transaction | 원자적 작업 단위 |

---

## 8. 학습 자료

- **Database Internals** — Alex Petrov (필독)
- **Designing Data-Intensive Applications** — Martin Kleppmann (필독)
- **Database System Concepts** — Silberschatz
- **High Performance MySQL** — Schwartz
- **PostgreSQL Documentation** — postgresql.org/docs (가장 좋은 docs)
- **Redis in Action** — Carlson
- **MongoDB: The Definitive Guide** — Bradshaw

---

## 9. 관련

- [[../computer-science/database-theory/database-theory]] — DB 이론 (관계 대수 / 정규화 / 동시성 이론)
- [[../computer-science/distributed-systems/distributed-systems]] — 분산 DB 의 토대 (Consensus, Replication)
- [[../backend/backend]] — backend framework 의 DB 연결
- [[../areas|↑ 20-areas]]
- [[../../00-inbox/links/links|↗ 외부 reference 카탈로그]]
