---
title: "Elasticsearch — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:05:00+09:00
tags:
  - database
  - elasticsearch
  - use-cases
---

# Elasticsearch — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 한 줄

> "**검색 / 로그 / 분석** 이 핵심이면 ES".

OLTP / 주력 저장소가 아니라 **보조 / 분석 계층**.

---

## 2. ✅ 적합

### 2.1 풀텍스트 검색
- 상품 / 콘텐츠 / 문서
- 한국어 nori
- 자동 완성 / suggest

### 2.2 로그 / APM / SIEM
- ELK / EFK 표준
- 비용은 ↑ — 대안 검토 (ClickHouse, Loki)

### 2.3 메트릭 / 시계열
- ILM rollover
- 압축 (best_compression)

### 2.4 Aggregation 분석
- facet (전자상거래)
- BI 대시보드 (Kibana)

### 2.5 Vector / Hybrid 검색
- KNN + 텍스트
- RAG 시스템

### 2.6 ML / Anomaly Detection
- Elastic ML
- Time series anomaly

### 2.7 Geo
- 위치 기반 검색

---

## 3. ⚠️ 검토

### 3.1 주력 저장소
- ES 는 검색 / 색인용 — 정합성 source of truth 는 RDB
- 일반 패턴: DB → CDC → ES

### 3.2 매우 자주 변경되는 데이터
- update 비용 ↑ (segment merge)
- 가능하면 batch / append-only

### 3.3 트랜잭션 / 일관성
- ACID X
- Near-real-time

### 3.4 작은 데이터셋
- PG full-text 가 충분할 수도
- 인프라 부담 정당화 X

---

## 4. ❌ 부적합

### 4.1 ACID 강한 도메인
- 금융 / 회계 — RDB

### 4.2 단순 KV
- Redis / DynamoDB

### 4.3 거대 OLAP (PB+)
- ClickHouse / BigQuery / Snowflake

### 4.4 임베디드
- SQLite

---

## 5. ES vs OpenSearch

| 항목 | Elasticsearch | OpenSearch |
| --- | --- | --- |
| 라이선스 | Elastic License | Apache 2.0 |
| AWS | Elastic Cloud | Amazon OpenSearch |
| Kibana | ✅ | OpenSearch Dashboards |
| 신규 기능 | 빠름 | 늦지만 따라옴 |
| Vector / ML | 강함 | 따라옴 |

AWS 환경 + 라이선스 자유 → OpenSearch.
신규 기능 + Kibana → Elasticsearch.

---

## 6. ES vs Solr

| 항목 | Elasticsearch | Solr |
| --- | --- | --- |
| 기반 | Lucene | Lucene |
| API | REST / JSON | REST / XML |
| 분산 | 자동 | ZooKeeper / SolrCloud |
| 인기 | ★★★★★ | ★★★ |
| 분석 / Aggregation | 강함 | facet OK |

신규 검색 시스템은 거의 ES.

---

## 7. ES vs Atlas Search / RediSearch

| 항목 | ES | Atlas Search | RediSearch |
| --- | --- | --- | --- |
| 외부 | ✅ 별도 시스템 | MongoDB 안 | Redis 안 |
| 통합 | 필요 | 통합 | 통합 |
| 풍부함 | ★★★★★ | ★★★★ | ★★★ |
| 한국어 | nori | Lucene 한국어 일부 | 일부 |

작은 / 통합 검색 → Atlas Search / RediSearch.
대규모 / 풍부 → ES.

---

## 8. ES vs ClickHouse / Druid

| 항목 | ES | ClickHouse | Druid |
| --- | --- | --- | --- |
| 주력 | 검색 + 분석 | OLAP | OLAP |
| 압축 | 보통 | 매우 좋음 | 좋음 |
| SQL | ESQL | 표준 | SQL-like |
| 풀텍스트 | ✅ 강함 | 보조 | 보조 |
| 운영 | 복잡 | 단순 | 복잡 |

로그 / 메트릭이 검색 + 분석 둘 다 필요 → ES.
순수 분석 (column scan) → ClickHouse.

---

## 9. ES vs Loki

Loki = Grafana 의 로그 시스템. 라벨 기반 + 풀텍스트 X.

- 단순 / 비용 ↓ → Loki
- 검색 / 분석 → ES

---

## 10. 결정 트리

```
풀텍스트 검색이 핵심?
└── Elasticsearch

로그 / 관측 + 검색?
├── 비용 우선 → Loki / ClickHouse
└── 풍부 → ES (ELK)

분석 (column scan) 거대?
└── ClickHouse / BigQuery

벡터 검색?
├── 작음 → pgvector / Redis vector
└── 큼 → ES KNN / Pinecone / Weaviate

자동 완성?
└── ES completion suggester

작은 검색 + RDB 있음?
└── PG FTS / pg_trgm 우선
```

---

## 11. 회사 사례

| 회사 | ES 사용 |
| --- | --- |
| GitHub | 코드 검색 |
| Wikipedia | 검색 |
| Netflix | 로그 |
| Uber | 로그 / 검색 |
| eBay | 검색 |
| Walmart | 검색 |
| Cisco | 보안 |
| Slack | 검색 |
| Yelp | 검색 |

---

## 12. 비용

ES 는 **돈이 많이 든다**:
- SSD + 충분한 RAM
- 매니지드 (Elastic Cloud / OpenSearch) 비용
- 라이선스 (Enterprise 기능)

→ 도입 전 ROI 점검. 단순 검색이면 PG full-text 도 검토.

---

## 13. 관련

- [[../mongodb/use-cases]] — Atlas Search
- [[../postgresql/use-cases]] — PG FTS / pgvector
- [[elasticsearch]] — ES hub
- [[../database]] — database hub
