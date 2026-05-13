---
title: "Cassandra — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:35:00+09:00
tags:
  - database
  - cassandra
  - use-cases
---

# Cassandra — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. 한 줄

> "**거대 쓰기 + 글로벌 분산 + 가용성** 우선이면 Cassandra".

기능은 적음 — **단순 KV-Wide-Column** + 분산 자동화 + 선형 확장.

---

## 2. ✅ 적합

### 2.1 거대 쓰기 (수만~수십만 TPS)
- LSM-Tree
- 쓰기 1 노드만 → 분산

### 2.2 시계열 / 로그 / 메트릭
- 시간 buckets + TWCS
- TTL 자동 만료
- 무한 append

### 2.3 IoT
- 거대 디바이스 + 영속

### 2.4 글로벌 다중 DC
- NetworkTopologyStrategy
- LOCAL_QUORUM 으로 강한 일관성 + 낮은 latency

### 2.5 사용자 활동 / 피드 / 메시지
- Netflix / Discord / Instagram 등

### 2.6 가용성 우선
- Masterless — single point of failure X

### 2.7 거대 카탈로그
- Apple iCloud 등

### 2.8 추천 / activity log
- Netflix recommendation engine

---

## 3. ⚠️ 검토

### 3.1 복잡 쿼리 / Aggregation
- ALLOW FILTERING X
- 옆에 Spark / Trino / Elasticsearch

### 3.2 강한 ACID
- LWT (Paxos) 가능 — 비싸다
- 진짜 강한 ACID 는 RDB

### 3.3 작은 데이터셋
- 운영 복잡도 비추
- RDB / Redis 가 더 단순

### 3.4 풀텍스트 검색
- SAI (5.0+) 일부
- 큰 검색은 ES

---

## 4. ❌ 부적합

### 4.1 강한 트랜잭션 (금융 거래)
- RDB / 분산 RDB (Spanner)

### 4.2 Ad-hoc 쿼리
- 쿼리 패턴 미정 → 모델링 불가

### 4.3 JOIN 많음
- denormalize 필수

### 4.4 작은 시스템
- 단일 노드 의미 X
- 3 노드 minimum

---

## 5. Cassandra vs ScyllaDB

| 항목 | Cassandra | ScyllaDB |
| --- | --- | --- |
| 언어 | Java | C++ (Seastar) |
| GC | 있음 | 없음 |
| Latency | ms | sub-ms |
| Throughput | 수만 TPS | 수십만 TPS |
| 호환성 | 표준 | Cassandra 호환 + 차이 |
| 라이선스 | Apache 2 | Source Available + Cloud |
| 운영 | 노드 ↑ | 노드 ↓ (성능 ↑) |

ScyllaDB = Cassandra 와 **같은 protocol + 더 빠름**. 신규는 Scylla 검토.

---

## 6. Cassandra vs DynamoDB

| 항목 | Cassandra | DynamoDB |
| --- | --- | --- |
| 운영 | 직접 (또는 Astra) | 완전 매니지드 |
| AWS 외 | 어디나 | AWS |
| 데이터 모델 | Wide Column + CQL | KV / Document |
| Consistency | tunable | eventual / strong |
| 비용 | 노드 비용 | 쓴 만큼 |
| Multi-region | NTS | Global Tables |

AWS 전용 + 매니지드 → DynamoDB.
멀티 클라우드 / 자체 운영 / CQL → Cassandra.

---

## 7. Cassandra vs HBase / Bigtable

| 항목 | Cassandra | HBase | Bigtable |
| --- | --- | --- | --- |
| 모델 | Wide Column | Wide Column | Wide Column |
| CAP | AP | CP | CP |
| 운영 | masterless | HMaster + ZK + HDFS | 매니지드 |
| GCP | X | X | ✅ |
| Hadoop | 옆에 | 통합 | X |

데이터 lake 통합 → HBase.
GCP 매니지드 → Bigtable.
일반 분산 → Cassandra / Scylla.

---

## 8. Cassandra vs MongoDB

| 항목 | Cassandra | MongoDB |
| --- | --- | --- |
| 모델 | Wide Column | Document |
| 분산 | Masterless | Replica Set + Sharding |
| 쓰기 확장 | 매우 강 | 강 (sharding) |
| Ad-hoc query | ❌ | ✅ |
| 운영 | 복잡 | 중간 |
| Consistency | tunable AP | CP 기본 |

Document + 쿼리 자유 → MongoDB.
거대 쓰기 + 글로벌 → Cassandra.

---

## 9. 결정 트리

```
거대 쓰기 (수만+ TPS)?
├── 글로벌 + Cassandra 호환 → ScyllaDB / Cassandra
├── AWS 전용 → DynamoDB
└── GCP → Bigtable

복잡 쿼리 / JOIN 많음?
└── RDB (PG / MySQL) 또는 MongoDB

작은 / 중간?
└── Cassandra 검토 X — RDB / Redis

시계열 / IoT?
└── Cassandra / Scylla / TimescaleDB

가용성 절대 우선?
└── Cassandra (AP)

강한 트랜잭션?
└── RDB / Spanner / Cockroach
```

---

## 10. 회사 사례

| 회사 | Cassandra 사용 |
| --- | --- |
| **Netflix** | 거대 fleet — 추천 / 사용자 데이터 |
| Apple | iCloud (거대 cluster) |
| Discord | 메시지 → Scylla 마이그 |
| Uber | 부분 |
| Instagram | 부분 |
| eBay | 부분 |
| Walmart | 부분 |
| Bloomberg | 시계열 |

→ Netflix 가 Cassandra 의 대표 사례. 수십만 노드까지 운영.

---

## 11. 운영 복잡도 주의

Cassandra 는 **운영이 복잡**:
- 모니터링 (JMX, Prometheus)
- 정기 Repair (Reaper)
- Compaction 튜닝
- Tombstone 관리
- Gossip / 네트워크 토폴로지

→ 데이터 규모 / 가용성 요구가 정당화되어야. 작은 시스템은 과대.

→ **매니지드 (Astra, ScyllaDB Cloud, AWS Keyspaces)** 가 운영 부담 ↓.

---

## 12. 관련

- [[../mongodb/use-cases]] — Document 비교
- [[../redis/use-cases]] — KV 비교
- [[../postgresql/use-cases]] — RDB 비교
- [[cassandra]] — Cassandra hub
- [[../database]] — database hub
