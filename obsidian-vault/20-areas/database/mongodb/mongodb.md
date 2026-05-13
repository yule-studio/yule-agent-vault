---
title: "MongoDB (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:20:00+09:00
tags:
  - database
  - nosql
  - document
  - mongodb
  - hub
---

# MongoDB (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 9 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**Document DB** — 데이터를 JSON-like (BSON) document 로 저장.
2009 10gen → MongoDB Inc. 가장 유명한 NoSQL document DB.
유연한 스키마 + 수평 확장 (sharding) + Aggregation Pipeline.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2007 | 10gen 설립 — 처음엔 PaaS |
| 2009 | MongoDB 1.0 (open source) |
| 2015 | 3.0 — WiredTiger 엔진 (기본화 3.2) |
| 2018 | **4.0** — Multi-document ACID transactions |
| 2018 | 라이선스 변경 (AGPL → SSPL) |
| 2019 | 4.2 — 분산 트랜잭션 |
| 2020 | Atlas 의 인기 → Cloud-first |
| 2024 | 8.0 — 성능 ↑, vector search GA |

---

## 3. MongoDB 의 특징

1. **Document Model** — BSON, 중첩 / 배열 / 다양 타입
2. **Schema flexibility** — 강제 X, 검증 옵션
3. **풍부한 쿼리** — 중첩 필드 / 배열 / 정규식 / aggregation
4. **Index** — B-Tree + Compound + Text + Geo + Wildcard
5. **Replication** — Replica Set (Raft 유사)
6. **Sharding** — 자동 수평 분할
7. **Aggregation Pipeline** — 강력한 분석
8. **Atlas** — 매니지드, vector search, 검색

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / mongosh / 첫 명령 |
| [[configuration]] | mongod.conf / 인증 / 리소스 |
| [[data-modeling]] | 임베드 vs 참조 / 패턴 |
| [[crud-syntax]] | find / insert / update / delete / 연산자 |
| [[indexes]] | 단일 / 복합 / Text / Geo / Wildcard / TTL |
| [[aggregation]] | Pipeline / $match / $group / $lookup / $facet |
| [[transactions]] | Multi-document ACID |
| [[replication]] | Replica Set / Election / Read Preference |
| [[sharding]] | shard key / chunk / balancer |
| [[security]] | --auth / role / 분실 복구 / 랜섬 사고 / 외부 노출 |
| [[use-cases]] | 언제 MongoDB 를 선택해야 하나 |

---

## 5. MongoDB 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 스키마 자주 변동 / 빠른 MVP | ✅ |
| 중첩 / 다양 타입의 document | ✅ |
| 콘텐츠 / 카탈로그 | ✅ |
| Event Sourcing / 로그 | ✅ |
| IoT / 시계열 (4.4+ time series) | ✅ |
| 검색 (Atlas Search) | ✅ |
| 벡터 검색 (8.0+) | ✅ |
| 강한 도메인 / JOIN 많음 | ⚠️ — PG + JSONB 도 검토 |
| 금융 / 강한 ACID | ⚠️ — 가능하지만 PG / Oracle |
| Graph traversal 깊음 | ❌ |
| 분석 (OLAP) | ❌ — ClickHouse / BigQuery |

---

## 6. 면접 핵심 질문

1. **Document vs Relational** — 트레이드오프.
2. **임베드 vs 참조** — 언제 어느 것.
3. **Aggregation Pipeline** 단계 / `$lookup`.
4. **Replica Set Election** — Raft.
5. **Read Preference / Read Concern / Write Concern**.
6. **Sharding** — shard key 선택.
7. **인덱스 종류** — compound prefix.
8. **트랜잭션 4.0+** — 한계 / 비용.
9. **`_id`** 와 ObjectId.
10. **WiredTiger** — MVCC / 압축.

---

## 7. 학습 자료

- **MongoDB Documentation** — mongodb.com/docs
- **MongoDB University** — university.mongodb.com (무료)
- **MongoDB: The Definitive Guide** (3rd)
- **Building on MongoDB** — patterns
- **Designing Data-Intensive Applications** Ch. 2-3

---

## 8. 관련

- [[../postgresql/postgresql]] — PG + JSONB 비교
- [[../redis/redis]] — KV
- [[../elasticsearch/elasticsearch]] — 검색
- [[../database|↑ database hub]]
