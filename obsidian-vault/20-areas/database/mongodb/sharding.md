---
title: "MongoDB Sharding — 수평 분할"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:05:00+09:00
tags:
  - database
  - mongodb
  - sharding
---

# MongoDB Sharding

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | shard key / chunk / balancer |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 한 줄

데이터를 여러 노드로 **수평 분할**. shard key 로 chunk 단위 분산. RAM / 디스크 / write throughput 모두 확장.

---

## 2. 구조

```
              ┌─────────────────────────┐
              │   mongos (라우터)         │
              │   (stateless, 다수)       │
              └────────┬────────────────┘
                       │
   ┌───────────────────┼───────────────────┐
   ▼                   ▼                   ▼
Shard 1 (RS)       Shard 2 (RS)       Shard 3 (RS)
                       │
              ┌────────┴────────┐
              │   Config Server  │
              │   (RS, 메타데이터)  │
              └─────────────────┘
```

### 2.1 컴포넌트

- **mongos** — 클라이언트가 접속. shard 메타 기반 라우팅.
- **Config Server** — chunk → shard 매핑 (Replica Set).
- **Shard** — 실제 데이터. 각 shard = Replica Set.

---

## 3. Shard Key

### 3.1 정의

```js
sh.shardCollection("app.orders", { userId: 1 })
sh.shardCollection("app.orders", { userId: "hashed" })

// Compound shard key
sh.shardCollection("app.events", { tenantId: 1, _id: 1 })
```

### 3.2 좋은 shard key

| 속성 | 의미 |
| --- | --- |
| **High Cardinality** | 값이 다양 (분산) |
| **Low Frequency** | 한 값에 데이터 집중 X |
| **Monotonic 피함** | `_id` 단조 증가 → hot shard |
| **쿼리에 자주 포함** | 라우팅 → 단일 shard hit |

### 3.3 나쁜 shard key

- `_id` 단조 (ObjectId) — 마지막 shard 만 write
- `timestamp` 같은 단조
- Low cardinality (예: `country: "KR"` 만 99%)

### 3.4 Hashed Shard Key

```js
sh.shardCollection("app.events", { userId: "hashed" })
```

- 균등 분포 보장
- 단점: 범위 쿼리 X (모든 shard)

### 3.5 Zone (옛 tag)

```js
sh.addShardTag("shard0", "kr")
sh.addShardTag("shard1", "us")
sh.addTagRange("app.users",
  { region: "KR", _id: MinKey }, { region: "KR", _id: MaxKey }, "kr")
```

지역별 데이터 위치 강제.

---

## 4. Chunk

shard key 의 범위 단위. 기본 128 MB (옛 64 MB).

```js
sh.status()        // chunk 분포
```

자동:
- 큰 chunk → **split**
- 불균등 → **migration** (balancer)

수동:
```js
sh.splitAt("app.orders", { userId: 100 })
sh.moveChunk("app.orders", { userId: 0 }, "shard1")
```

---

## 5. Balancer

```js
sh.startBalancer()
sh.stopBalancer()
sh.getBalancerState()

// 시간 창
sh.setBalancerState(true)
db.settings.update(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "02:00", stop: "06:00" } } },
  { upsert: true }
)
```

운영 시간엔 끄고 새벽에 마이그.

---

## 6. 쿼리 라우팅

### 6.1 Targeted Query — 단일 shard

```js
db.orders.find({ userId: 123 })   // shard key 포함 → 1 shard
```

### 6.2 Scatter-gather — 모든 shard

```js
db.orders.find({ status: "completed" })   // shard key 없음 → 모두
```

→ shard key 를 거의 모든 쿼리에 포함시키는 것이 핵심.

### 6.3 Compound shard key 의 prefix

```
{ tenantId: 1, _id: 1 }

✅ targeted
{ tenantId: 1 }
{ tenantId: 1, _id: 100 }

❌ scatter
{ _id: 100 }
```

---

## 7. 클러스터 설정

### 7.1 Config Server (RS)

```yaml
sharding:
  clusterRole: configsvr
replication:
  replSetName: configRS
```

### 7.2 Shard (RS)

```yaml
sharding:
  clusterRole: shardsvr
replication:
  replSetName: shard1
```

### 7.3 mongos

```yaml
sharding:
  configDB: configRS/cfg1:27019,cfg2:27019,cfg3:27019
```

### 7.4 클러스터 만들기

```js
// 한 mongos 에서
sh.addShard("shard1/s1a:27018,s1b:27018,s1c:27018")
sh.addShard("shard2/s2a:27018,s2b:27018,s2c:27018")

// 데이터베이스 sharding 활성
sh.enableSharding("app")

// 컬렉션 sharding
sh.shardCollection("app.orders", { userId: 1 })

sh.status()
```

---

## 8. Resharding (5.0+)

```js
db.adminCommand({
  reshardCollection: "app.orders",
  key: { tenantId: 1, _id: 1 }
})

db.adminCommand({ shardingState: 1 })
```

shard key 변경. 옛 버전은 dump+restore.

---

## 9. 운영

### 9.1 모니터링

```js
sh.status()
sh.status({ verbose: true })
db.collection.getShardDistribution()

db.changelog.find().sort({ time: -1 }).limit(10)  // 마이그 이력
```

### 9.2 mongos 상태

```js
db.runCommand({ connectionStatus: 1 })
db.runCommand({ getShardMap: 1 })
```

---

## 10. 트랜잭션 + Sharding

4.2+ 부터 cross-shard 트랜잭션 — 2PC.

```js
// 비용 ↑ — 가능한 single-shard 트랜잭션
session.startTransaction()
// 모든 collection 의 shard key 가 같은 값이면 1 shard
db.users.update({tenantId:"A", _id:1}, ...)
db.orders.insert({tenantId:"A", ...})
session.commitTransaction()
```

---

## 11. 백업

| 방법 | 일관성 |
| --- | --- |
| Per-shard mongodump | shard 별 — 일관성 X |
| Balancer stop + 모든 shard snapshot | 일관성 ✅ |
| Atlas Snapshot | 자동 |
| Ops Manager Backup | 자동 |

---

## 12. Sharded vs Non-sharded — 결정

| 시나리오 | 권장 |
| --- | --- |
| RAM 충분 + write 적당 | Replica Set |
| Working set > RAM | Shard 검토 |
| Write throughput > 단일 노드 | Shard |
| 글로벌 분산 / 지역 데이터 | Shard + Zone |

⚠️ Shard 는 운영 복잡도 ↑. 한계 도달 전엔 미리 도입 X.

---

## 13. 함정

### 13.1 Hot Shard (단조 shard key)
`_id` (ObjectId) 처럼 시간 증가 = 마지막 shard 만 사용. **Hashed** 또는 다른 키.

### 13.2 Scatter-gather 쿼리
shard key 없는 쿼리. 모든 shard hit — 성능 한계. shard key 를 쿼리에 포함.

### 13.3 Jumbo Chunk
한 key 에 데이터 폭증 — split 불가. shard key 잘못 선택.

### 13.4 cross-shard 트랜잭션 남용
2PC 비용. single-shard 트랜잭션 설계.

### 13.5 mongos 의 connection 수
각 mongos → 모든 shard 에 connection. 폭증 — pool 크기 신중.

### 13.6 Config Server 의 RS 미구성
단일 노드 = 메타데이터 SPOF. RS 필수 (3.4+).

### 13.7 Resharding 의 비용
대용량 = 시간 + I/O 폭증. 운영 시간 외.

### 13.8 Balancer 가 운영에 영향
큰 chunk 마이그 = 디스크 / 네트워크. 야간 창.

---

## 14. 매니지드 — Atlas

Atlas 의 sharding 은 클릭 한 번. 단, shard key 선택은 여전히 사용자 책임 — 가장 중요한 결정.

---

## 15. 학습 자료

- **MongoDB Sharding** — docs.mongodb.com/manual/sharding
- **Choosing a Shard Key** — MongoDB blog
- **MongoDB University** — M103/M201

---

## 16. 관련

- [[replication]] — RS = shard 의 토대
- [[data-modeling]] — shard key 와 모델링
- [[mongodb]] — MongoDB hub
