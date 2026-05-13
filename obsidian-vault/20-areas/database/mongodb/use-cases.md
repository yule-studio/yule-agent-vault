---
title: "MongoDB — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:10:00+09:00
tags:
  - database
  - mongodb
  - use-cases
---

# MongoDB — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 한 줄

> "**스키마가 자주 변하거나 자연스럽게 document** 일 때 MongoDB".

PG + JSONB 도 강력하지만, document 가 핵심이고 빠른 변화가 필요하면 MongoDB.

---

## 2. ✅ 적합

### 2.1 빠른 MVP / Prototype
- 스키마 미정착
- 빠른 변경

### 2.2 콘텐츠 / 카탈로그
- 상품 / 미디어 / 글 — 자연스러운 document
- 변동 많은 속성

### 2.3 Event Sourcing / 로그
- append-only
- 시계열 collection (5.0+)

### 2.4 IoT / 시계열
- Time Series Collection
- 압축 / aggregation

### 2.5 모바일 / 게임 백엔드
- 사용자 프로필 — 다양한 속성
- 빠른 변화

### 2.6 검색 (Atlas Search)
- Atlas 의 Lucene 기반
- 한 DB 안에서 OLTP + Search

### 2.7 AI / 벡터 (Atlas Vector Search, 8.0)
- 메타데이터 + 임베딩
- 같은 DB

### 2.8 SaaS 멀티테넌트
- tenant 마다 다른 속성
- 컬렉션 / database 분리도 가능

### 2.9 Aggregation 중심 분석
- 풍부한 pipeline
- $facet 으로 동시 분석

---

## 3. ⚠️ 검토

### 3.1 강한 도메인 / JOIN 많음
- PG + JSONB 가 RDB 의 안정성 + JSON 유연성
- $lookup 은 비쌈

### 3.2 강한 ACID 트랜잭션
- 4.0+ multi-document 가능
- 비싸고 한계 있음

### 3.3 OLAP / 대량 분석
- ClickHouse / BigQuery 가 우위
- Atlas Data Lake / Federation 으로 확장 가능

### 3.4 Full-text search 매우 큰 규모
- Atlas Search OK, 거대 / 복잡 = Elasticsearch

### 3.5 KV / 캐시
- Redis

---

## 4. ❌ 부적합

### 4.1 임베디드 / 모바일
- SQLite

### 4.2 Graph traversal 깊음
- Neo4j

### 4.3 단순 RDB 워크로드
- 도메인이 관계형 명확 — PG / MySQL

---

## 5. MongoDB vs PostgreSQL

자세히 → [[../postgresql/use-cases#6-postgresql-vs-mongodb]]

| 항목 | MongoDB | PostgreSQL |
| --- | --- | --- |
| 스키마 | 자유 + validator | 강 + JSONB |
| 관계 | 임베드 / $lookup | JOIN 표준 |
| ACID | 4.0+ | 표준 |
| 확장 | Sharding 내장 | Citus / 외부 |
| 운영 | RS 복잡 | 단일 노드 단순 |
| 한국어 검색 | Atlas Search | 외부 (ES) |
| 벡터 | Atlas | pgvector |
| 매니지드 | Atlas | RDS / Aurora / Supabase |

요약: **도메인 안정 + 관계 많음 → PG**. **스키마 빠른 변화 + Document 자연 → MongoDB**.

---

## 6. MongoDB vs DynamoDB / Cosmos DB

| 항목 | MongoDB | DynamoDB |
| --- | --- | --- |
| API | MongoDB | Key-value + secondary index |
| 쿼리 | 풍부 | 제한적 |
| 트랜잭션 | multi-doc | 제한 |
| 운영 | RS / Shard | 완전 매니지드 |
| 비용 | 보통 | 사용한 만큼 |

DynamoDB = AWS 전용 + 매우 단순 모델 + 무한 확장.
Cosmos DB = MongoDB API 호환 (일부) + multi-model.

---

## 7. MongoDB vs Couchbase / Elastic

| 항목 | MongoDB | Couchbase | Elasticsearch |
| --- | --- | --- | --- |
| 주력 | OLTP Document | Document + KV + Search | Search |
| 인기 | ★★★★★ | ★★★ | ★★★★★ (검색) |

검색 위주 → ES. 일반 document → MongoDB. KV 캐시 + 일부 query → Couchbase.

---

## 8. 결정 트리

```
주력 저장소?
├── Document 자연 + 변화 잦음 → MongoDB
├── 관계 중심 / SQL 친숙 → PostgreSQL
└── KV / 캐시 → Redis

검색 중심?
└── Elasticsearch / Atlas Search

OLAP?
└── ClickHouse / BigQuery

거대 쓰기 / 분산 KV?
└── Cassandra / DynamoDB
```

---

## 9. 회사 사례

| 회사 | MongoDB 사용 |
| --- | --- |
| Adobe | Creative Cloud |
| Forbes | CMS |
| Toyota | 일부 |
| Verizon | 일부 |
| Trello | 카드 / 보드 |
| eBay | 일부 |
| Cisco | 일부 |
| Bosch | IoT |
| Sega | 게임 데이터 |

---

## 10. Atlas 의 가치

Atlas (매니지드) 가 MongoDB 의 강점:
- Replica Set / Sharding 자동
- 백업 / 복원
- Atlas Search (Lucene)
- Atlas Vector Search (8.0)
- Atlas Data Lake / Federation
- Atlas Stream Processing
- 멀티 클라우드 / 리전

→ Atlas 라면 운영 부담 ↓↓. 신규 MongoDB 는 Atlas 가 표준.

---

## 11. 관련

- [[../postgresql/use-cases]] — PG 비교
- [[../redis/use-cases]] — Redis 비교
- [[mongodb]] — MongoDB hub
- [[../database]] — database hub
