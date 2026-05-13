---
title: "Elasticsearch / OpenSearch (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:20:00+09:00
tags:
  - database
  - search
  - elasticsearch
  - opensearch
  - hub
---

# Elasticsearch / OpenSearch (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 9 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**분산 검색 / 분석 엔진**. Lucene 기반. JSON 문서 + 역색인 (inverted index) → μs~ms 검색.
ELK / EFK 스택의 E. 로그 / 검색 / APM / SIEM 의 표준.

---

## 2. Elasticsearch vs OpenSearch

| 항목 | Elasticsearch | OpenSearch |
| --- | --- | --- |
| 시작 | 2010 (Elastic) | 2021 fork (AWS) |
| 라이선스 | Elastic License (8.x), SSPL → 다시 OSI 호환 (8.16+) | Apache 2.0 |
| AWS 매니지드 | Elastic Cloud | Amazon OpenSearch Service |
| Kibana | Elastic | OpenSearch Dashboards |

→ 7.10 까지는 둘 다 호환. 이후 분기. **AWS 환경 = OpenSearch**, **기능 / Kibana = Elasticsearch**.

---

## 3. 역사

| 연도 | 사건 |
| --- | --- |
| 2010 | Elasticsearch 1.0 (Shay Banon) |
| 2012 | Elastic 회사 설립 |
| 2015 | ELK Stack 인기 |
| 2018 | 6.0 — Term Vector / removed _type |
| 2020 | 7.x — 단일 type, security 무료 |
| 2021 | **라이선스 변경 (SSPL)** → AWS **OpenSearch fork** |
| 2022 | 8.0 — KNN / Vector search |
| 2024 | 8.x — ESQL, AGG 성능 ↑ |

---

## 4. 특징

1. **역색인** — token → posting list
2. **분산 / 샤딩** — 자동
3. **Near Real Time** — refresh 후 검색 (기본 1초)
4. **Rich Query DSL** — bool, match, term, range, agg
5. **Aggregation** — facet / 집계 / 분석
6. **Analyzer** — 한국어 nori, 영어 standard, 중국어 SmartCN
7. **Vector Search** — KNN (8.0+)
8. **ELK** — Logstash / Beats / Kibana

---

## 5. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / Kibana / 첫 인덱스 |
| [[configuration]] | elasticsearch.yml / JVM / 디스크 |
| [[index-mapping]] | mapping / field type / dynamic |
| [[query-dsl]] | bool / match / term / range / 검색 |
| [[aggregations]] | metric / bucket / pipeline |
| [[analyzer-korean]] | analyzer / nori / synonym |
| [[cluster-shards]] | shard / replica / 노드 역할 |
| [[performance-tuning]] | refresh / merge / cache / 메모리 |
| [[use-cases]] | 언제 ES 를 선택해야 하나 |

---

## 6. ES 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 풀텍스트 검색 | ✅ 표준 |
| 로그 / 메트릭 (ELK) | ✅ 표준 |
| APM / SIEM | ✅ |
| 자동 완성 / 추천 | ✅ |
| 한국어 검색 (nori) | ✅ |
| Aggregation 분석 | ✅ |
| Vector / Hybrid 검색 (8.0+) | ✅ |
| 주력 OLTP | ❌ — RDB |
| ACID 트랜잭션 | ❌ |
| 작은 데이터 + 단순 LIKE | ⚠️ — RDB FTS 가능 |

---

## 7. 면접 핵심 질문

1. **역색인 (Inverted Index)** 의 구조.
2. **shard vs replica**.
3. **mapping** 의 중요성 / dynamic mapping 함정.
4. **bool query** — must / should / filter / must_not.
5. **analyzer** 의 동작 — tokenizer / filter.
6. **한국어 검색** — nori / 명사 추출.
7. **Aggregation** — terms / date histogram / cardinality.
8. **Refresh / Flush / Merge**.
9. **ILM (Index Lifecycle Management)**.
10. **Vector / KNN search**.

---

## 8. 학습 자료

- **Elasticsearch Definitive Guide** (오래됐지만 개념 좋음)
- **Elasticsearch in Action** — 2nd ed.
- **Elasticsearch Documentation** — elastic.co/guide
- **OpenSearch Documentation** — opensearch.org/docs
- **Elastic Blog** — elastic.co/blog

---

## 9. 관련

- [[../mongodb/mongodb]] — Atlas Search 비교
- [[../postgresql/postgresql]] — FTS 비교
- [[../redis/redis]] — RediSearch 비교
- [[../database|↑ database hub]]
