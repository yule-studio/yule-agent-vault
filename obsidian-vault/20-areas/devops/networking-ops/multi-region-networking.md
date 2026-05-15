---
title: "Multi-region networking — DR / latency / cost"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:49:00+09:00
tags: [devops, networking-ops, multi-region, dr]
---

# Multi-region networking — DR / latency / cost

**[[networking-ops|↑ networking-ops]]**

---

## 1. 왜 multi-region

```
1. DR (Disaster Recovery)
   - 한 region down → 다른 region 으로
   
2. latency
   - 사용자 가까운 region 서비스
   
3. compliance
   - data residency (EU 의 data 는 EU 안만)
   
4. capacity / cost
   - region 별 capacity 또는 spot 가용성
```

→ 시니어 = trade-off 분석 + 구현.

---

## 2. multi-region 패턴

| | 무엇 |
| --- | --- |
| **Single region** | 1 region 만 (기본) |
| **Multi-AZ in 1 region** | 같은 region 의 다른 AZ — DR within region |
| **Active-Passive** | primary + DR (failover 시만 활성) |
| **Active-Active** | 모든 region 가 traffic |
| **Active-Active + geo** | region 별 사용자 다름 |
| **Cell-based** | 작은 isolated cell (★ AWS pattern) |

---

## 3. data layer 결정 (★ 가장 어려움)

```
A. single primary + multi-region read replica
   - write = 1 region
   - read = N region (eventual consistency)
   - failover = 새 primary promote
   - 예: Postgres + replica, DynamoDB Global Tables

B. multi-master
   - 모든 region 가 write
   - conflict resolution 필요 (CRDT / LWW)
   - 예: Cassandra, Couchbase, Cosmos DB
   
C. geo-partitioned
   - 사용자 별 home region
   - "내 데이터" 는 한 region 만
   - 예: Spanner geo-partitioning

D. distributed transaction
   - 모든 region 가 동시 commit
   - 매우 비싸 / 느림
   - 예: Spanner (TrueTime), CockroachDB
```

→ application 요구 → 한 모델.

---

## 4. data replication 도구

```
DB:
  - Postgres logical / streaming replication
  - MySQL group replication / Vitess
  - DynamoDB Global Tables
  - Aurora Global Database
  - Spanner / Cosmos DB
  - CockroachDB / TiDB

cache:
  - Redis Active-Active (CRDT, Enterprise)
  - 또는 region 별 cold cache

object:
  - S3 Cross-Region Replication
  - GCS Multi-Region bucket

search:
  - Elasticsearch cross-cluster replication
  - OpenSearch

queue:
  - Kafka MirrorMaker 2
  - SQS / SNS cross-region 없음 (manual)
```

---

## 5. network 연결

```
between AWS regions:
  - Transit Gateway peering
  - VPC peering (region 간)
  - Direct Connect Gateway

between cloud:
  - Megaport / Equinix
  - VPN (slow)
  - SD-WAN

backbone:
  - AWS Global Network (Global Accelerator 의 backbone)
  - GCP private backbone
  - Cloudflare (Magic Transit)
```

---

## 6. latency 측정 (★)

```
AWS region 간 latency 표 (대략):
  us-east-1 ↔ us-west-2:   60-70ms
  us-east-1 ↔ ap-northeast-2: 150-180ms
  ap-northeast-2 ↔ ap-southeast-1: 70-90ms
  us-east-1 ↔ eu-west-1:    70-90ms
  
빛의 속도 의 한계:
  서울 ↔ 뉴욕: 약 11,000 km / c = 37ms (이론 최소)
  실제: 150-200ms (network hop)

→ 같은 region 의 AZ 간 = 1-2ms.
→ 다른 region = synchronous call X (latency overhead).
```

→ Cloudping.co / AWS CloudPing 으로 측정.

---

## 7. traffic routing (★)

```
A. DNS based (Route53 latency routing)
   - 사용자 → 가까운 region
   - DNS cache delay
   - 단순 / 흔함

B. GCP Global LB / AWS Global Accelerator (anycast)
   - 한 IP, BGP routing
   - 빠른 failover
   - 비용 ↑

C. CDN (Cloudflare / CloudFront)
   - edge POP
   - origin = 한 region 만
   - static / cached content

D. application-level routing
   - login 후 user_id 기반 region 결정
   - sticky session
```

---

## 8. cost 함정 (★)

```
cross-region traffic:
  AWS: $0.02/GB (in/out)
  → 1 TB 의 replication = $20
  → 1 PB / mo = $20,000

data 의 cross-region cost:
  - Cross-Region Replication (S3): $0.02/GB
  - Aurora Global: replication cost + storage in both
  - DynamoDB Global Tables: write × N region
  - CloudWatch Logs cross-region: 비쌈

avoidance:
  - region-local primary (가능하면)
  - asynchronous batch (cheaper)
  - selective replication (critical 만)
  - data summarization (raw 안 보내고 aggregate)
```

---

## 9. RTO / RPO 매트릭스

| Strategy | RTO | RPO | Cost |
| --- | --- | --- | --- |
| Backup only | 24h+ | hours-days | $ |
| Pilot Light | 1-4h | minutes | $$ |
| Warm Standby | 10-30min | seconds | $$$ |
| Active-Active | <1min | 0-seconds | $$$$ |

→ business 의 요구 + budget.

---

## 10. failover automation

```
manual failover:
  1. 사람이 incident 인지
  2. DR runbook 따라
  3. DB promote / DNS 변경 / Pod 시작
  4. 검증

automated failover:
  1. health check fail
  2. Route53 / Global Accelerator 가 자동 switch
  3. DB 자동 promote (Aurora Global, RDS Multi-AZ)
  4. Pod 미리 standby

→ automated 가 좋지만 false positive 위험.
   대부분 = automated detection + manual decision.
```

---

## 11. multi-region 의 hidden cost

```
1. cross-region data transfer ($$$)
2. duplicate infra (region 별 server)
3. cross-region DB replication (compute + storage)
4. licensing (per-CPU 라이센스)
5. monitoring / logging (region 별 metric)
6. compliance (data residency 가 더 비쌈)
7. operational complexity (사람 시간)
8. test / staging (어디까지 multi-region?)
```

→ 일반: 1 region active + multi-AZ + S3 backup to second region.

→ 큰 회사: cell-based architecture.

---

## 12. cell-based architecture (★ AWS)

```
한 region 의 service 를:
  - 작은 cell 들로 분할
  - 각 cell = isolated stack (DB / cache / pod)
  - 사용자 = 특정 cell 에 mapping
  - 한 cell fail = 일부 user 영향 (전체 X)

장점:
  - blast radius 작음
  - 점진 deploy (cell 별)
  - autoscale 단위
  - failure isolation
  
운영 부담 ↑.
큰 SaaS / AWS service 자체가 cell-based.
```

---

## 13. data residency / sovereignty

```
EU 의 GDPR:
  - EU 사용자 data → EU region 만
  - non-EU 도 EU 에 저장 OK

China / Russia:
  - 별도 cloud (Alibaba / Yandex)
  - 또는 hybrid

US 정부:
  - GovCloud (AWS / Azure 별도 region)

→ 사용자 별 region 결정 + tag 표시.
```

---

## 14. CDN + multi-region origin

```
fallback chain:
  user → CDN POP (cache hit?)
       → CDN POP (cache miss) → primary origin
       → 만약 primary fail → secondary origin
       
benefits:
  - 대부분 traffic = CDN (origin 부담 ↓)
  - origin 만 multi-region (작은 비용)
  - CDN 의 anycast 흡수
```

---

## 15. 실전 흐름 (★)

```
Phase 1 (start):
  - 1 region active, multi-AZ
  - S3 cross-region backup
  - DR runbook

Phase 2 (growth):
  - 2nd region pilot light
  - DB cross-region replica (async)
  - DNS failover (manual)

Phase 3 (mature):
  - 2nd region warm standby
  - automated failover
  - read replica per region

Phase 4 (global):
  - active-active
  - geo-partitioning
  - CDN + edge compute
  - cell-based
```

---

## 16. 함정

1. **multi-region 무조건** — 작은 회사 부적합 (비용 / complexity).
2. **DR drill 안 함** — 실제 disaster 시 fail.
3. **synchronous cross-region** — latency 가 user 영향.
4. **cost projection 무시** — egress 폭증.
5. **data residency 위반** — 법적 영향.
6. **failover automation 의 false positive** — 의도치 않은 cutover.
7. **secondary region 의 stale config** — drift.

---

## 17. 관련

- [[networking-ops|↑ networking-ops]]
- [[load-balancer-types]]
- [[bgp-anycast]]
- [[../sre/disaster-recovery|↗ DR]]
- [[../finops/architecture-optimization|↗ egress cost]]
