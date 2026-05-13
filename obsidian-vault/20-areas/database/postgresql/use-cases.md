---
title: "PostgreSQL — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:00:00+09:00
tags:
  - database
  - postgresql
  - use-cases
---

# PostgreSQL — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 시나리오 + 비교 |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 한 줄 요약

> "**의심스러우면 PostgreSQL**".

대부분의 워크로드를 잘 처리. 단점이 명확한 케이스에만 다른 DB 검토.

---

## 2. ✅ 강력 추천

### 2.1 일반 웹 / API 서비스 OLTP
- 도메인 모델 명확 / JOIN 많음 / 일관성 필요
- 사용자 / 주문 / 결제 / 권한

### 2.2 SaaS 멀티테넌트
- Row Level Security (RLS) 로 tenant 격리
- Schema-per-tenant 도 가능

### 2.3 JSON 중심 + 관계형 일부
- `JSONB` 로 유연 + 관계형 안전
- MongoDB 후보를 PostgreSQL 로 대체

### 2.4 지리정보
- **PostGIS** 는 사실상 표준
- Google Maps 백엔드도 PostGIS 기반 시기 존재

### 2.5 AI / 벡터 검색
- **pgvector** — 같은 DB 안에서 메타데이터 + 임베딩
- Pinecone / Weaviate 대안

### 2.6 시계열
- **TimescaleDB** — hypertable, continuous aggregate
- IoT, 메트릭

### 2.7 전통적 DW / OLAP (소~중)
- 병렬 쿼리, BRIN, Window
- 거대해지면 Citus / ClickHouse 검토

### 2.8 Event Sourcing / CQRS
- WAL + Logical replication = CDC
- Debezium → Kafka 표준

### 2.9 GIS / 위치 기반 서비스
- 음식 배달 / 차량 / 부동산

---

## 3. ⚠️ 조건부 — 검토 필요

### 3.1 거대 OLAP (수 TB+)
- **ClickHouse / BigQuery / Snowflake** 가 우위
- PG 는 columnar 가 약함 (Citus / pg_columnar 검토)

### 3.2 거대 쓰기 (수만 TPS+)
- Cassandra / ScyllaDB
- PG 는 sharding 외부 (Citus / pg_dog)

### 3.3 풀텍스트 검색 (대규모)
- 작으면 PG full-text + pg_trgm OK
- 크면 Elasticsearch / Meilisearch

### 3.4 단순 KV / 캐시
- Redis 가 압도적
- PG 는 더 무거움

### 3.5 그래프 (소셜, 추천)
- 깊은 traversal — Neo4j, Memgraph
- 얕으면 PG 의 CTE 로 가능

### 3.6 IoT / 거대 시계열
- TimescaleDB 로 어느 정도 — 더 큰 규모면 InfluxDB, ClickHouse

---

## 4. ❌ 부적합

### 4.1 임베디드 / 모바일 / 데스크탑 단일 파일
- **SQLite** — 단일 .db 파일, 서버 X

### 4.2 인메모리 캐시
- **Redis / Memcached**

### 4.3 분산 / 글로벌 / 멀티 리전 + 다중 마스터
- CockroachDB / Spanner / YugabyteDB

### 4.4 글로벌 read low latency
- 글로벌 CDN 처럼 — 위는 분산 DB

---

## 5. PostgreSQL vs MySQL

| 항목 | PostgreSQL | MySQL |
| --- | --- | --- |
| 표준 SQL 준수 | ★★★★★ | ★★★ |
| 트랜잭션 DDL | ✅ | ❌ (대부분) |
| 동시성 | MVCC | InnoDB MVCC |
| 인덱스 종류 | 6+ | B-Tree, FT, Hash, Spatial |
| JSON | JSONB (강력) | JSON 함수 |
| 복잡 쿼리 / Window / CTE | ★★★★★ | ★★★★ (8.0+) |
| 단순 쓰기 / 읽기 | 비슷 | 비슷 |
| 복제 다양성 | Streaming + Logical | Statement / Row + Group |
| 운영 노하우 | 깊지만 적음 | 광범위 |
| 클라우드 매니지드 | 풍부 | 풍부 |
| 단순함 / 입문 | 보통 | 쉬움 |

요약: 복잡한 도메인 / 정확성 → PostgreSQL. 단순 / 읽기 위주 / 거대 운영 노하우 필요 → MySQL.

자세히 → [[../mysql/use-cases]]

---

## 6. PostgreSQL vs MongoDB

| 항목 | PostgreSQL | MongoDB |
| --- | --- | --- |
| 스키마 | 강 (변경 가능) | 약 (유연) |
| JSON 저장 | JSONB | BSON |
| 인덱스 | 다양 | 다양 |
| 트랜잭션 | ACID 강 | 4.0+ 다중 문서 ACID |
| 조인 | 강력 | `$lookup` (느림) |
| 스키마 진화 | migration | 자유 |
| 운영 복잡도 | 낮음 (단일) | 높음 (replica set) |
| 학습 곡선 | SQL | 자체 문법 |

요약: 도메인 모델 안정 → PG. 스키마 자주 변동 / Document-heavy → Mongo. 그러나 **대부분의 경우 PG + JSONB 로 충분**.

---

## 7. PostgreSQL vs SQLite

| 항목 | PostgreSQL | SQLite |
| --- | --- | --- |
| 형태 | 서버 | 임베디드 (파일) |
| 동시 쓰기 | 다수 | 1 (writer 락) |
| 크기 | 무제한 | ~ 1TB 까지 가능 |
| 운영 | 필요 | 0 |
| 용도 | 운영 백엔드 | 모바일 / 데스크탑 / 테스트 |

---

## 8. PostgreSQL vs CockroachDB / YugabyteDB

| 항목 | PostgreSQL | CockroachDB |
| --- | --- | --- |
| 프로토콜 | PG wire | PG wire 호환 |
| 분산 | 외부 (Citus) | 내장 |
| Multi-region | 외부 | 내장 |
| 일관성 | 단일 노드 강 | 분산 + Raft |
| 단점 | 단일 노드 한계 | 단일 노드 성능 낮음 / 비용 ↑ |

여러 리전 / 다중 마스터 필요 → CockroachDB. 그 외엔 PG.

---

## 9. 결정 트리

```
스키마 명확, JOIN 필요?
├── 예 → PostgreSQL
└── 아니오 →
    KV / 캐시?
    ├── 예 → Redis
    └── 아니오 →
        Document 자유?
        ├── 예 → MongoDB (또는 PG + JSONB)
        └── 아니오 →
            검색 중심?
            ├── 예 → Elasticsearch
            └── 아니오 →
                거대 쓰기 분산?
                ├── 예 → Cassandra
                └── 아니오 → PostgreSQL (의심스러우면)
```

---

## 10. 마이그레이션 시그널

### 10.1 MySQL → PostgreSQL
- 복잡 분석 쿼리 / Window 가 필요
- JSON / 지리 / 벡터 필요
- 트랜잭션 DDL / 표준 SQL 절실

### 10.2 MongoDB → PostgreSQL
- 도메인 모델이 안정되어 스키마 관리가 필요
- JOIN / 트랜잭션 / 일관성 비중 ↑
- JSONB 로 충분히 흡수

### 10.3 PostgreSQL → 다른 DB
- 단일 노드 한계 (수만 TPS+) → Cassandra / Scylla
- OLAP 거대 → ClickHouse / BigQuery
- Multi-region 강요 → CockroachDB

---

## 11. 회사 사례

| 회사 | PostgreSQL 사용 |
| --- | --- |
| Instagram | 처음부터 PG (지금도) |
| Reddit | 핵심 PG + 분산 |
| Stripe | 메인 PG |
| GitLab | 메인 PG |
| Apple | 거대 PG fleet |
| Robinhood | 메인 PG |

---

## 12. 관련

- [[../mysql/use-cases]] — MySQL 비교
- [[../mongodb/use-cases]] — MongoDB 비교
- [[../sqlite/use-cases]] — SQLite 비교
- [[postgresql]] — PostgreSQL hub
- [[../database]] — database hub
