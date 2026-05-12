---
title: "database — DB 실무 운영"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - index
  - database
---

# database — DB 실무 운영

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 폴더링 골격 |

**[[../areas|↑ 20-areas]]**

> RDB / NoSQL / 검색 엔진 별 운영. 이론은 `computer-science/database-theory`, backend 통합은 `backend/<framework>/db-connection.md`.

## 하위 영역

| 주제 | 진입점 | 한 줄 |
| --- | --- | --- |
| PostgreSQL | [[postgresql/postgresql]] | 인덱스 / EXPLAIN / extension / replication |
| MySQL | [[mysql/mysql]] | InnoDB / replication / Aurora |
| Redis | [[redis/redis]] | data types / pub-sub / persistence / 분산 락 |
| MongoDB | [[mongodb/mongodb]] | aggregation / sharding / Atlas |
| Elasticsearch / OpenSearch | [[elasticsearch/elasticsearch]] | 검색 / ELK / Korean nori |
| SQLite | [[sqlite/sqlite]] | 경량 / 로컬 / 모바일 |
| Cassandra / ScyllaDB | [[cassandra/cassandra]] | 분산 NoSQL |
| DB 설계 실무 | [[database-design/database-design]] | 정규화 / 인덱스 전략 / migrations |
| DB 튜닝 | [[database-tuning/database-tuning]] | 쿼리 최적화 / 통계 / pg_stat / EXPLAIN |

## 관련
- [[../areas|↑ 20-areas]]
- [[../00-inbox/links/links|↗ 외부 reference 카탈로그]]