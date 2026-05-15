---
title: "CAP / PACELC — consistency 결정"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:37:00+09:00
tags: [devops, distributed-systems, cap]
---

# CAP / PACELC — consistency 결정

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. CAP (★ 1999 Brewer)

```
3 중 2 만 선택:
  C - Consistency:    모든 read 가 최신 write 또는 error
  A - Availability:   모든 request 응답 (오래 걸려도)
  P - Partition tolerance: network 끊김에도 동작
```

```
P 는 분산에서 사실상 필요 → C 또는 A 선택:
  
CP: partition 발생 시 일부 node 가 거부 (단 모순 X)
AP: partition 발생 시 모두 응답 (단 stale 가능)

CA: partition 없는 가정 = single-node 또는 LAN
   현실 분산 ≠ CA
```

---

## 2. 흔한 잘못된 이해

```
❌ "CP 면 항상 consistent + partition tolerant"
✅ "partition 시 unavailable, normal 시 consistent"

❌ "CA = consistency + availability"
✅ "partition 없을 때만. partition 시 CA = 둘 다 잃음"

❌ "AP 면 항상 available + partition tolerant"
✅ "partition 시 available, 단 inconsistent 가능"
```

→ CAP 은 "partition 시 행동" 이론.

---

## 3. PACELC (★ Daniel Abadi, 2010)

CAP 의 보완 — partition 외에도:

```
if Partition:
    choose A (Availability) or C (Consistency)
else (Normal):
    choose L (Latency) or C (Consistency)
    
→ partition 없어도 trade-off 존재.
```

| 시스템 | partition 시 | normal 시 | 의미 |
| --- | --- | --- | --- |
| **Dynamo / Cassandra** | AP | EL | 항상 availability + latency 우선 |
| **Riak** | AP | EL | |
| **BigTable / HBase** | CP | EC | 항상 consistency |
| **Spanner** | CP | EC | TrueTime + Paxos |
| **MongoDB** | CP | EL or EC | replica 설정 따라 |
| **DynamoDB** | CP (strong) or AP (eventual) | EL or EC | client 가 선택 |

---

## 4. 실전 결정

### read your writes 필요 — primary read

```
A user 가 post 작성 → 즉시 본다.

방법:
  - primary 에서 read (replica lag 무시)
  - sticky session (user → 같은 replica)
  - write 후 short cache
  - causal consistency
```

### 검색 / 추천 — eventual OK

```
"좋아요 100 vs 99" → 사용자 신경 X.

방법:
  - read replica (lag 1-10s OK)
  - cache 적극
  - eventual consistent DB (Cassandra)
```

### 결제 — strong

```
잔액 → 정확해야.

방법:
  - 한 primary 만 write
  - SELECT FOR UPDATE
  - distributed transaction (2PC) 또는 Saga
  - idempotency key
```

---

## 5. consistency model spectrum (★)

```
strict ──── strong ──── seq ──── causal ──── eventual ──── stale
가장 strict ────────────────────────── 가장 weak
가장 비싸 ──────────────────────────── 가장 cheap
```

### Strong (linearizable)

```
모든 client 의 모든 read = 같은 순서.

guarantees:
  - 한 변수의 read 가 가장 최근 write 반환
  - 모든 client 가 같은 view

가격: latency / availability.

예: Spanner, ZooKeeper, etcd.
```

### Sequential

```
한 process 의 operation 순서 보존.
다른 process 와는 reorder 가능.

예: Kafka (per-partition).
```

### Causal

```
인과 관계 보존:
  A → B 면 모두가 그 순서로 봄.
  
예: Twitter timeline, COSMOS.
```

### Eventual

```
update 멈춤 → 결국 모두 같음.
잠시 stale OK.

예: Cassandra, DynamoDB (eventual mode), DNS.
```

### Read-your-writes

```
본인 write 즉시 본인은 본다.
다른 user 는 잠시 stale.

방법: primary read after write.
```

### Monotonic reads

```
시간 역행 X.
한 번 새 값 → 옛 값 안 봄.

방법: sticky session.
```

---

## 6. Dynamo style (quorum)

```
N = 3 replica
W = write 가 success 으로 인정할 노드 수
R = read 가 검증할 노드 수

W + R > N  ⇒ strong consistency

흔한:
  N=3, W=2, R=2  → strong
  N=3, W=1, R=1  → eventual (빠름)
  N=3, W=3, R=1  → read 빠름 / write 느림
```

→ trade-off application 별 선택.

---

## 7. partition 발생 시

```
3-node cluster (A, B, C):
  network 가 (A,B) vs (C) 로 split

CP system:
  C 가 minority → C 는 read/write reject (split-brain 방지)
  A,B 가 quorum → 동작
  → C 의 client = unavailable

AP system:
  C 도 응답 → 자체 write 받음
  나중 합쳐질 때 conflict → CRDT 또는 LWW (last-write-wins)
  → 일시적 inconsistent
```

---

## 8. 실전 anti-pattern

```
❌ "AP DB 인데 strong 필요"
  → application 에서 lock / sync
  → 잘못된 도구 선택

❌ "CP DB 의 read replica"
  → replica 가 AP 같이 동작 (stale)
  → strong 가정 깨짐

❌ "single-node = no partition"
  → 데이터 center 와 service 간도 partition
  → application 의 retry / circuit 필요

❌ "eventual 면 consistency 없어도 된다"
  → 결국 일치 + monotonic / read-your-writes 도 필요
```

---

## 9. CAP 의 한계

```
실제 시스템 = spectrum.
"이 op 는 strong, 저 op 는 eventual" mix 흔함.

예: DynamoDB:
  - default eventual read
  - ConsistentRead=true 옵션 → strong
  
예: MongoDB:
  - writeConcern: majority / 1
  - readPreference: primary / secondary
```

→ "이 시스템은 X" 보다 "이 op 는 X" 가 정확.

---

## 10. 시니어 결정 매트릭스

```
data:
  - 사용자 요구 (즉시 보여야?)
  - 비즈니스 영향 (잘못된 값의 cost)
  - 사용자 multi-region?
  
도구:
  - C 가 critical = Spanner / Postgres + sync replication
  - A 가 critical = Cassandra / Dynamo (AP)
  - mix = primary write + replica read
  - 단순 = single Postgres + cache (cache stale OK)
  
test:
  - chaos (network partition 시뮬)
  - load (replication lag 측정)
  - failover (RTO/RPO 검증)
```

---

## 11. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[replication]]
- [[consensus-raft]]
- [[../performance/database-performance|↗ DB performance]]
