---
title: "Elasticsearch Cluster — Shard / Replica / 노드 역할"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:55:00+09:00
tags:
  - database
  - elasticsearch
  - cluster
---

# Elasticsearch Cluster

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | shard / replica / 노드 |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 한 줄

ES 의 데이터는 **인덱스 → 샤드 → Lucene 인덱스** 로 분산. shard = 단위 작업 / 복제 / 라우팅.

---

## 2. 인덱스 / Shard / Replica

```
Index "products"
  ├── Primary Shard 0  (Node A)
  │     └── Replica Shard 0  (Node B)
  ├── Primary Shard 1  (Node B)
  │     └── Replica Shard 1  (Node C)
  └── Primary Shard 2  (Node C)
        └── Replica Shard 2  (Node A)
```

- **Primary** — 쓰기 받음, replica 로 전파
- **Replica** — 읽기 / 가용성
- **Primary + Replica** 가 같은 노드에 X (안전)

---

## 3. Shard 의 의미

각 shard = **하나의 Lucene 인덱스**. shard 안에서 검색이 일어남.

쿼리:
```
Query → coordinator (random 노드)
       → 모든 primary or replica 중 하나에 흩뿌림
       → 결과 모아서 정렬 / merge
       → 클라이언트
```

---

## 4. shard 개수 결정

생성 시 고정 (`number_of_shards`) — **변경 X**.

### 4.1 가이드

- 한 shard = 10~50 GB 권장
- 너무 작으면 메타데이터 부담
- 너무 크면 단일 노드 부담 / 이동 비용 ↑

### 4.2 작은 인덱스
`number_of_shards: 1` — 충분.

### 4.3 큰 인덱스 / 시계열
ILM rollover 로 매일/주 별 분할.

---

## 5. Replica 개수

`number_of_replicas` — 동적 변경 가능.

```http
PUT /products/_settings
{ "index": { "number_of_replicas": 2 } }
```

- 0 = 가용성 X (dev)
- 1 = 표준
- 2+ = 읽기 분산 ↑, 디스크 × N

---

## 6. 노드 역할 — node.roles

| Role | 의미 |
| --- | --- |
| `master` | 클러스터 상태 |
| `data` | 데이터 (all tiers) |
| `data_hot`, `data_warm`, `data_cold`, `data_frozen` | 데이터 tier |
| `data_content` | 비-시계열 데이터 |
| `ingest` | ingest pipeline |
| `ml` | ML |
| `transform` | transform |
| `remote_cluster_client` | CCR / CCS |
| `[]` (empty) | coordinating-only |

### 6.1 작은 클러스터
3 노드 모두 `[master, data, ingest]`.

### 6.2 큰 클러스터
- 3 dedicated master
- N data
- 일부 coordinating-only (검색 부담)

---

## 7. Master Election

3+ master-eligible 노드. 과반수 (quorum) 로 선출.

```yaml
cluster.initial_master_nodes: [ "m1", "m2", "m3" ]   # 첫 부트만
```

⚠️ 짝수 = split brain 위험. 항상 홀수.

---

## 8. Allocation

새 shard 의 노드 배치. 정책:
- 디스크 사용량
- shard 분포 균등
- shard awareness (rack, zone)
- 사용자 지정 attribute

```yaml
node.attr.zone: a
cluster.routing.allocation.awareness.attributes: zone
```

```http
PUT /products/_settings
{
  "index.routing.allocation.include.zone": "a,b"
}
```

---

## 9. Tier — Hot/Warm/Cold/Frozen

```yaml
node.roles: [ data_hot ]   # 또는 data_warm, data_cold, data_frozen
```

| Tier | 디스크 | 용도 |
| --- | --- | --- |
| Hot | NVMe | 최신 색인 + 검색 |
| Warm | SSD | 최근 검색 |
| Cold | HDD | 거의 안 읽음 |
| Frozen | Object storage | 거의 X — searchable snapshot |

ILM 이 자동 마이그.

---

## 10. ILM (Index Lifecycle Management)

```http
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "50gb", "max_age": "1d" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "cold":   { "min_age": "30d", "actions": { "freeze": {} } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

### 10.1 Data Stream (7.9+)

```http
PUT /_index_template/logs
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": { "index.lifecycle.name": "logs-policy" }
  }
}

POST /logs/_doc        // → logs-000001, 000002, ... 자동
{ "@timestamp": "...", "message": "..." }
```

→ 시계열에 표준.

---

## 11. Snapshot / Restore

```http
PUT /_snapshot/repo { ... }   // S3 등록

PUT /_snapshot/repo/snap-2026-05-13
{ "indices": "logs-*" }

POST /_snapshot/repo/snap-2026-05-13/_restore
```

### 11.1 Searchable Snapshot (Enterprise)
콜드 데이터를 S3 에서 직접 검색. 비용 ↓.

---

## 12. Cross-Cluster Replication / Search

### 12.1 CCR (Enterprise)
한 클러스터 → 다른 클러스터로 인덱스 복제. DR.

### 12.2 CCS (무료)
한 쿼리로 여러 클러스터 검색.

```http
GET /cluster1:logs-*,cluster2:logs-*/_search
```

---

## 13. 모니터링

### 13.1 cat APIs

```http
GET /_cat/health?v
GET /_cat/nodes?v
GET /_cat/shards?v
GET /_cat/indices?v
GET /_cat/allocation?v
GET /_cat/recovery?v
GET /_cat/pending_tasks
GET /_cat/thread_pool?v
```

### 13.2 cluster status

| 색 | 의미 |
| --- | --- |
| green | 모든 primary + replica OK |
| yellow | primary OK, replica 일부 X |
| red | primary 일부 X — 데이터 손실 가능 |

```http
GET /_cluster/health
GET /_cluster/state
GET /_cluster/stats
GET /_cluster/pending_tasks
GET /_cluster/allocation/explain
```

### 13.3 Unassigned shard 해결

```http
GET /_cluster/allocation/explain
```

원인 표시 — 디스크 / 노드 down / awareness.

---

## 14. Reroute / Cancel

```http
POST /_cluster/reroute
{
  "commands": [
    { "move": { "index": "logs-2026-05-13", "shard": 0,
                "from_node": "node1", "to_node": "node2" } },
    { "cancel": { "index": "logs-...", "shard": 0, "node": "node1" } }
  ]
}
```

운영 미세 조정. 자동 allocation 이 보통 충분.

---

## 15. 함정

### 15.1 너무 많은 shard
shard 메타데이터 부담. 보통 노드당 600 shard / GB heap 1 이하.

### 15.2 너무 큰 shard
shard 이동 / 복구 비용 ↑. 100GB 초과 피함.

### 15.3 짝수 master
split brain. 홀수 (3, 5).

### 15.4 단일 노드 + replica 1
1 replica unassigned → yellow. 0 또는 노드 추가.

### 15.5 운영 중 mapping 변경
일부만 가능. text analyzer 변경은 reindex.

### 15.6 deleted document 누적
Lucene 의 마킹 → force_merge 로 정리.

```http
POST /logs-2026-04/_forcemerge?max_num_segments=1
```

### 15.7 unassigned shard 방치
red 상태 — `allocation/explain` 으로 원인.

### 15.8 cluster.initial_master_nodes 유지
첫 부트 후 제거. 안 그러면 새 노드 join 이상.

---

## 16. 학습 자료

- **Elasticsearch Cluster Reference**
- **Shard sizing recommendations** — elastic blog
- **ILM Documentation**

---

## 17. 관련

- [[configuration]] — 노드 / 디스크
- [[performance-tuning]] — 튜닝
- [[elasticsearch]] — ES hub
