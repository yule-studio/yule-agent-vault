---
title: "Azure Cache for Redis"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:00:00+09:00
tags:
  - azure
  - database
  - redis
  - cache
---

# Azure Cache for Redis

**[[database|↑ Database]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

매니지드 Redis — AWS ElastiCache / GCP Memorystore 동등. Enterprise tier = RedisJSON / RediSearch / Active-Geo Replication.

---

## 2. Tier

| Tier | 의미 |
| --- | --- |
| Basic | single node, dev |
| Standard | master + replica |
| Premium | clustering + persistence |
| Enterprise | Redis Enterprise (RedisJSON, RediSearch 등 모듈) |

---

## 3. 사용

```bash
az redis create -g myapp -n myapp-cache -l koreacentral \
  --sku Standard --vm-size c1

az redis show -g myapp -n myapp-cache --query hostName
```

```hcl
resource "azurerm_redis_cache" "cache" {
  name                = "myapp-cache"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  capacity            = 1
  family              = "C"
  sku_name            = "Standard"
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
  }
}
```

---

## 4. 접속

```python
import redis
r = redis.Redis(
    host="myapp-cache.redis.cache.windows.net",
    port=6380,
    password="...",
    ssl=True,
)
r.set("key", "value")
```

→ TLS port 6380 (non-SSL 6379 disable 권장).

---

## 5. Redis 자체 사용법

자세히 → [[../../../database/redis/redis|↗ Redis hub]]

---

## 6. 비용 (대략)

| SKU | $/월 |
| --- | --- |
| C1 Standard (1 GB) | ~$60 |
| C2 Standard (2.5 GB) | ~$120 |
| P1 Premium (6 GB) | ~$450 |

---

## 7. 함정

- TLS 강제 (port 6380)
- Premium 만 VNet
- max client 한계 (size 별)
- persistence Premium / Enterprise 만

---

## 8. 관련

- [[database]]
- [[../../../database/redis/redis|↗ Redis 자체]]
