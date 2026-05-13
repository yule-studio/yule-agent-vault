---
title: "Cassandra / ScyllaDB (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:00:00+09:00
tags:
  - database
  - nosql
  - wide-column
  - cassandra
  - scylla
  - hub
---

# Cassandra / ScyllaDB (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 7 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**분산 Wide-Column NoSQL**. masterless (peer-to-peer) + LSM-Tree + tunable consistency.
2008 Facebook → Apache → DataStax. 거대 쓰기 / 선형 확장 / 다중 DC 의 표준.

**ScyllaDB** = C++ rewrite, 같은 protocol + 더 빠름.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2007 | Facebook 의 Inbox Search 위해 개발 (Avinash Lakshman, Prashant Malik) |
| 2008 | 오픈소스 |
| 2010 | Apache top-level |
| 2014 | DataStax 의 상용화 |
| 2014 | **ScyllaDB** — C++ port (Seastar 프레임워크) |
| 2018 | 4.0 — Virtual table, Materialized view 시험 |
| 2021 | 4.1 — performance |
| 2024 | 5.0 — 새 storage engine, tablets |

---

## 3. Cassandra 의 특징

1. **Masterless** — 모든 노드 동등
2. **LSM-Tree** — 쓰기 최적
3. **Tunable Consistency** — 쓰기 / 읽기마다 조절
4. **Linear Scalability** — 노드 추가 = 성능 ↑
5. **Multi-DC** — 지역 복제 표준
6. **AP** (CAP) — 가용성 우선
7. **CQL** — SQL 비슷한 쿼리 언어
8. **Wide Row** — 한 partition 안 거대 컬럼

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / cqlsh / 첫 명령 |
| [[configuration]] | cassandra.yaml / JVM / 노드 |
| [[data-modeling]] | partition key / clustering / denormalize |
| [[cql-syntax]] | CREATE / INSERT / SELECT / LWT |
| [[consistency-tunable]] | CL / Quorum / Gossip / Hinted Handoff |
| [[replication-strategy]] | SimpleStrategy / NetworkTopology / Multi-DC |
| [[use-cases]] | 언제 Cassandra 를 선택해야 하나 |

---

## 5. Cassandra 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 거대 쓰기 (수만~수십만 TPS) | ✅ 표준 |
| 시계열 / 로그 / 메트릭 | ✅ |
| IoT / 센서 | ✅ |
| 글로벌 다중 DC | ✅ |
| 거대 카탈로그 (Netflix recommendations) | ✅ |
| 사용자 활동 / 피드 | ✅ |
| 강한 ACID 트랜잭션 | ❌ — RDB |
| 복잡 JOIN / Aggregation | ❌ — RDB / OLAP |
| 작은 시스템 | ❌ — 운영 복잡 과대 |
| 풀텍스트 검색 | ⚠️ — ES |

---

## 6. 면접 핵심 질문

1. **Wide Column** 모델 — partition / clustering key.
2. **Consistency Level** — ONE / QUORUM / ALL.
3. **CAP** — Cassandra 는 AP 인가 CP 인가.
4. **Gossip / Snitch / Replication strategy**.
5. **Compaction** — STCS / LCS / TWCS.
6. **Hinted Handoff** / Read Repair / Anti-entropy.
7. **Tombstone** — DELETE 와 GC.
8. **LWT** (Lightweight Transaction) — Paxos.
9. **Materialized View / Secondary Index** 의 한계.
10. **Bloom filter / SSTable** 구조.

---

## 7. 학습 자료

- **Cassandra Documentation** — cassandra.apache.org/doc
- **Cassandra: The Definitive Guide** (3rd) — Hewitt
- **DataStax Academy** — academy.datastax.com (무료)
- **ScyllaDB Documentation** — docs.scylladb.com
- **Designing Data-Intensive Applications** Ch. 5-6

---

## 8. 관련

- [[../mongodb/mongodb]] — Document 비교
- [[../redis/redis]] — KV 비교
- [[../database|↑ database hub]]
