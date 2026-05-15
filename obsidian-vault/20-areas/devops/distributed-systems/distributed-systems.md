---
title: "Distributed systems — CAP / consensus / patterns ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:35:00+09:00
tags: [area, devops, distributed-systems]
---

# Distributed systems — CAP / consensus / patterns ★

**[[../devops|↑ devops]]**

---

## 1. 왜 시니어가 알아야

```
MSA / cloud / replicated DB / cache / queue:
  → 모두 분산 시스템

시니어 결정:
  - 데이터 어디 둘 건가 (consistency)
  - replica 의 lag 허용?
  - leader 선출?
  - 정확한 once delivery?
  - distributed transaction?
  - failure mode?
```

→ "한 server" 의 사고와 다름.

---

## 2. fallacies of distributed computing (★)

> Sun Microsystems / Peter Deutsch — 흔한 잘못된 가정 8가지:

```
1. The network is reliable.        — 끊김 항상 발생
2. Latency is zero.                — 네트워크 시간
3. Bandwidth is infinite.          — 제한
4. The network is secure.          — 침해 가능
5. Topology doesn't change.        — 변경됨
6. There is one administrator.     — multi-team
7. Transport cost is zero.         — 비용
8. The network is homogeneous.     — 다양 환경
```

→ 분산 design 의 출발점.

---

## 3. CAP theorem

```
P (Partition tolerance) — 분산이면 항상 필요
  → 선택: C 또는 A

CP: Consistency 우선 — partition 시 reject 또는 wait
   예: 은행 / 결제 / leader election
   
AP: Availability 우선 — partition 시 stale 응답
   예: 검색 / 추천 / 콘텐츠

→ 실제로는 PACELC (partition 시 + normal 시)
```

---

## 4. PACELC (★ 정확)

```
if partition (P):
   then choose A vs C
   
else (no partition):
   then choose L (latency) vs C (consistency)
   
예:
  Dynamo / Cassandra : AP / EL — 가용성 + 저 latency
  Spanner           : CP / EC — consistency 항상
  MongoDB           : CP / EL
  PostgreSQL       : CA / EC (single node)
```

---

## 5. consistency model

| | 무엇 |
| --- | --- |
| **Strong (linearizable)** | 모든 client 가 같은 순서 | 비싸 |
| **Sequential** | order 일관 (각 client) | |
| **Causal** | 인과관계 보존 | |
| **Eventual** | 결국 일치 | 빠름 |
| **Read your writes** | 본인 write 즉시 본다 | |
| **Monotonic reads** | 시간 역행 안 | |

→ application 요구 vs 비용 tradeoff.

---

## 6. 하위 영역

- [[cap-pacelc]] — consistency 모델 깊이
- [[consensus-raft]] — Raft / Paxos / Zab
- [[saga-pattern]] — distributed transaction 대안
- [[event-sourcing-cqrs]] — event 기반 / read write 분리
- [[replication]] — sync / async / quorum
- [[sharding-partitioning]] — 데이터 분산
- [[idempotency]] — exactly-once / dedup
- [[circuit-breaker]] — fail-fast / fallback
- [[backpressure]] — overload 대응
- [[leader-election]] — 단일 의사결정자
- [[distributed-lock]] — 분산 락 (Redlock / etcd)
- [[messaging-patterns]] — at-least-once / outbox / dead-letter
- [[time-clocks]] — wall clock vs logical clock (Lamport / vector)
- [[failure-modes]] — split-brain / Byzantine / network partition
- [[pitfalls]]

---

## 7. 학습 순서

1. Day 1: [[cap-pacelc]] + [[consistency-models]]
2. Day 2: [[replication]] + [[sharding-partitioning]]
3. Day 3: [[consensus-raft]] + [[leader-election]] + [[distributed-lock]]
4. Day 4: [[saga-pattern]] + [[event-sourcing-cqrs]] + [[idempotency]]
5. Day 5: [[circuit-breaker]] + [[backpressure]] + [[failure-modes]]

---

## 8. 추천 도서 (★)

- **Designing Data-Intensive Applications** (Kleppmann) — bible
- **Database Internals** (Petrov)
- **Patterns of Distributed Systems** (Joshi)
- **Release It!** (Nygard) — circuit / bulkhead
- **Microservices Patterns** (Richardson) — saga
- **Building Microservices** (Newman)

---

## 9. 시스템 별 사례

| | consistency | 시스템 |
| --- | --- | --- |
| 은행 잔액 | strong | PostgreSQL master |
| 좋아요 count | eventual | Cassandra / Redis |
| inventory | strong (lock) | DB FOR UPDATE / distributed lock |
| 검색 index | eventual | Elasticsearch (replication lag) |
| user profile | read-your-writes | RDS replica + read-after-write |
| 추천 | eventual | offline batch |
| 채팅 message | causal | Kafka order by partition |

---

## 10. 관련

- [[../devops|↑ devops]]
- [[../../60-recipes/spring/architecture/architecture|↗ architecture]]
- [[../networking-ops/service-mesh|↗ service mesh]]
- [[../performance/database-performance|↗ DB perf]]
