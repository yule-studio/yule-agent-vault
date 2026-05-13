---
title: "Elasticsearch 시작하기 — 설치 / Kibana / 첫 인덱스"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:25:00+09:00
tags:
  - database
  - elasticsearch
  - setup
---

# Elasticsearch 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / Kibana / 첫 명령 |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 설치

### 1.1 Docker (가장 쉬움)

```bash
docker network create elastic

docker run -d --name es01 --net elastic \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms2g -Xmx2g" \
  docker.elastic.co/elasticsearch/elasticsearch:8.13.0

docker run -d --name kib01 --net elastic \
  -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=http://es01:9200 \
  docker.elastic.co/kibana/kibana:8.13.0
```

### 1.2 macOS

```bash
brew tap elastic/tap
brew install elastic/tap/elasticsearch-full
brew services start elastic/tap/elasticsearch-full
```

### 1.3 매니지드

| 서비스 | 특징 |
| --- | --- |
| **Elastic Cloud** | 공식 매니지드 |
| **Amazon OpenSearch Service** | AWS 매니지드 |
| **Bonsai** | 작은 호스팅 |
| **Aiven** | 멀티 클라우드 |

---

## 2. 첫 호출

```bash
curl http://localhost:9200
# { "version": { "number": "8.13.0" }, "tagline": "You Know, for Search" }

curl http://localhost:9200/_cluster/health
curl http://localhost:9200/_cat/nodes?v
curl http://localhost:9200/_cat/indices?v
```

### 2.1 Kibana Dev Tools
브라우저 `http://localhost:5601` → Dev Tools → Console 에서 REST 명령. **운영 도구로 매우 강력**.

---

## 3. 인덱스 / 문서 — REST

### 3.1 인덱스 생성

```http
PUT /products
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "name":   { "type": "text", "analyzer": "standard" },
      "price":  { "type": "double" },
      "stock":  { "type": "integer" },
      "tags":   { "type": "keyword" },
      "createdAt": { "type": "date" }
    }
  }
}
```

### 3.2 문서 색인

```http
POST /products/_doc
{ "name": "Apple", "price": 1.5, "stock": 100, "tags": ["fruit"] }

PUT /products/_doc/1
{ "name": "Banana", "price": 0.5, "stock": 200, "tags": ["fruit"] }

POST /products/_create/1     // 없을 때만
{ ... }
```

### 3.3 조회

```http
GET /products/_doc/1
GET /products/_search
GET /products/_search
{
  "query": { "match": { "name": "apple" } }
}
```

### 3.4 수정

```http
POST /products/_update/1
{
  "doc": { "price": 0.6 }
}

POST /products/_update_by_query
{
  "query": { "term": { "tags": "fruit" } },
  "script": { "source": "ctx._source.price *= 1.1" }
}
```

### 3.5 삭제

```http
DELETE /products/_doc/1
DELETE /products
```

### 3.6 Bulk

```http
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "Apple", "price": 1.5 }
{ "index": { "_index": "products" } }
{ "name": "Banana" }
{ "delete": { "_index": "products", "_id": "5" } }
```

각 줄 끝에 newline 필수.

---

## 4. _cat APIs — 빠른 진단

```http
GET /_cat/health?v
GET /_cat/nodes?v
GET /_cat/indices?v&s=index
GET /_cat/shards?v
GET /_cat/aliases?v
GET /_cat/templates?v
GET /_cat/thread_pool?v
GET /_cat/allocation?v
GET /_cat/recovery?v
GET /_cat/master
```

---

## 5. 보안 (8.0+)

8.0 부터 **기본 보안 활성**. 시작 시 superuser 비밀번호 / enrollment token 생성.

```bash
# 비밀번호 재설정
bin/elasticsearch-reset-password -u elastic

# API key
curl -u elastic:pass -X POST "http://localhost:9200/_security/api_key" -H 'Content-Type: application/json' -d '
{ "name": "my-api-key", "expiration": "1d" }
'
```

```http
PUT /_security/role/products_ro
{
  "indices": [
    { "names": ["products*"], "privileges": ["read"] }
  ]
}

POST /_security/user/alice
{
  "password": "secret",
  "roles": ["products_ro"]
}
```

---

## 6. 클라이언트

```python
# Python
from elasticsearch import Elasticsearch
es = Elasticsearch("http://localhost:9200", api_key="...")
es.index(index="products", document={"name": "A"})
es.search(index="products", query={"match": {"name": "apple"}})
```

```javascript
// Node
import { Client } from '@elastic/elasticsearch'
const c = new Client({ node: 'http://localhost:9200', auth: { apiKey: '...' } })
await c.index({ index: 'products', document: {...} })
```

라이브러리:
- Python — `elasticsearch` (공식)
- Node — `@elastic/elasticsearch`
- Java — Java API Client (8.0+)
- Go — `go-elasticsearch`

---

## 7. JVM Heap — 가장 중요

```bash
# config/jvm.options 또는 env
ES_JAVA_OPTS="-Xms16g -Xmx16g"
```

규칙:
- RAM 의 **50%**
- **32 GB 이하** (compressed oops)
- min = max

---

## 8. 함정

### 함정 1 — Dynamic mapping
첫 색인 시 자동 추정. 잘못된 타입 고정 → 명시 mapping 권장.

### 함정 2 — `text` vs `keyword`
text = 분석 (검색), keyword = 그대로 (필터 / 집계). 잘못 쓰면 비효율.

### 함정 3 — 인증 없이 노출
8.0 이전 / 비활성한 경우. 즉시 침해. 인터넷 노출 금지.

### 함정 4 — Heap > 32GB
compressed oops 해제 → 오히려 느림.

### 함정 5 — 너무 많은 작은 인덱스
shard 폭증 → 메타데이터 부담. 컨솔리데이션 / ILM rollover.

### 함정 6 — Refresh 1초 가정
색인 → 즉시 검색 X. `?refresh=true` 또는 1초 대기.

---

## 9. 관련

- [[configuration]] — elasticsearch.yml
- [[index-mapping]] — 매핑
- [[query-dsl]] — 쿼리
- [[elasticsearch]] — ES hub
