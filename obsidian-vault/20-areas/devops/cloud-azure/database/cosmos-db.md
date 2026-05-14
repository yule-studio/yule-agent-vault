---
title: "Azure Cosmos DB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:55:00+09:00
tags:
  - azure
  - database
  - cosmos-db
  - nosql
---

# Azure Cosmos DB

**[[database|↑ Database]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

**글로벌 분산 multi-model NoSQL** — Document / Key-Value / Graph / Column / Mongo API / Cassandra API / PostgreSQL (Citus 기반).

5 가지 일관성 모델 (strong → eventual). 자동 multi-region replication.

---

## 2. API

| API | 의미 |
| --- | --- |
| **NoSQL** (Core SQL) | native — JSON document |
| **MongoDB** | Mongo wire compatible |
| **Cassandra** | CQL |
| **Gremlin** | Graph |
| **Table** | KV (Azure Table 호환) |
| **PostgreSQL** | Citus 분산 PG |

→ 신규 = NoSQL (Core). 이식 = Mongo / Cassandra API.

---

## 3. 사용

```bash
az cosmosdb create -g myapp -n myapp-cosmos \
  --kind GlobalDocumentDB \
  --locations regionName=koreacentral failoverPriority=0 \
  --default-consistency-level Session

az cosmosdb sql database create -g myapp -a myapp-cosmos -n app
az cosmosdb sql container create -g myapp -a myapp-cosmos -d app -n users \
  --partition-key-path /userId --throughput 400
```

```hcl
resource "azurerm_cosmosdb_account" "main" {
  name                = "myapp-cosmos"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  consistency_policy {
    consistency_level = "Session"
  }

  geo_location {
    location          = "koreacentral"
    failover_priority = 0
  }
  geo_location {
    location          = "japaneast"
    failover_priority = 1
  }

  capabilities { name = "EnableServerless" }   # serverless 모드
}
```

---

## 4. 일관성 5 단계

| Level | 의미 |
| --- | --- |
| Strong | linearizable (single-region) |
| Bounded staleness | K version 또는 T time lag 까지 |
| Session (default) | 자기 write 의 read |
| Consistent prefix | 순서 보장 only |
| Eventual | 가장 약함 |

→ default Session 이 가장 흔함.

---

## 5. 가격 모델

### 5.1 Provisioned RU/s

```
1 RU = 1 KB doc read
1 KB doc write ~ 5 RU
$0.008 / 100 RU·시간
```

### 5.2 Serverless

```
$0.25 / 1M RU
50 GB storage 한계
auto scale 없음
```

### 5.3 Autoscale
max RU 설정 — burst 시만 비용 ↑.

---

## 6. SDK

```python
from azure.cosmos import CosmosClient
client = CosmosClient(url, credential=token)
db = client.get_database_client("app")
container = db.get_container_client("users")

container.upsert_item({"id":"alice", "userId":"alice", "email":"a@x.com"})
item = container.read_item("alice", partition_key="alice")
items = list(container.query_items("SELECT * FROM c WHERE c.email = @e",
                                     parameters=[{"name":"@e","value":"a@x.com"}],
                                     enable_cross_partition_query=True))
```

---

## 7. Partition Key

- 한 partition = 20 GB 한계
- key 잘못 = hot partition / cross-partition query 비용

자세히 → [[../../../database/dynamodb/dynamodb]] (비슷한 원리)

---

## 8. Change Feed

DynamoDB Streams 동등 — 변경 이벤트 → Function / Stream.

---

## 9. 비용 함정

- RU 폭증 — query 가 cross-partition
- 큰 doc — RU per write 비례
- 여러 region — replication cost

---

## 10. 관련

- [[database]]
- [[../../../database/mongodb/mongodb]] — Mongo API 호환
- [[../../../database/cassandra/cassandra]] — Cassandra API
