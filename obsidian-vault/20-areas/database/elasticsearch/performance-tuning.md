---
title: "Elasticsearch 성능 튜닝 — Refresh / Merge / Cache / 메모리"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:00:00+09:00
tags:
  - database
  - elasticsearch
  - performance
---

# Elasticsearch 성능 튜닝

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 색인 / 검색 / 메모리 |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 메모리

### 1.1 Heap = RAM × 50%, ≤ 32 GB

자세히 → [[configuration#4-jvm--jvmoptions]]

### 1.2 OS 캐시 = 나머지

Lucene 파일이 mmap. 검색 성능의 큰 부분.

### 1.3 indices.memory.index_buffer_size

```yaml
indices.memory.index_buffer_size: 10%
```

색인 버퍼 — 큰 색인 부하 시 늘릴 수 있음.

---

## 2. 색인 성능

### 2.1 Bulk API 사용

```http
POST /_bulk
{ "index": { "_index": "logs" } }
{ ... }
{ "index": { "_index": "logs" } }
{ ... }
```

- 5~15 MB 또는 1000~5000 doc 권장
- 한 번에 너무 크면 메모리 ↑, 너무 작으면 round-trip

### 2.2 Replica 0 으로 임시 색인

```http
PUT /products/_settings
{ "index": { "number_of_replicas": 0 } }

// 대량 색인 후
PUT /products/_settings
{ "index": { "number_of_replicas": 1 } }
```

### 2.3 refresh_interval 늘림

```http
PUT /logs/_settings
{ "index": { "refresh_interval": "30s" } }
```

기본 1초 → 30초 / 60초 (또는 `-1` 임시) → 색인 처리량 ↑.

⚠️ 변경 후 검색 가시성 지연.

### 2.4 translog flush

```yaml
index.translog.durability: async    # 매 sync 안 함
index.translog.sync_interval: 30s
```

`request` (기본) vs `async`. async 면 crash 시 30 초 손실 가능.

### 2.5 _id 자동 생성

```http
POST /logs/_doc           // ✅ 자동 _id — 빠름
{ ... }

PUT /logs/_doc/12345      // 사용자 _id — 검증 비용
```

대량 색인 시 자동 ID 권장.

### 2.6 코덱

```http
PUT /logs
{
  "settings": {
    "index.codec": "best_compression"   // 또는 default
  }
}
```

`best_compression` = zstd / deflate → 디스크 ↓, CPU ↑.

---

## 3. Merge / Segment

Lucene 의 작은 segment 들이 합쳐짐.

```yaml
index.merge.scheduler.max_thread_count: 1     # SSD 면 OK, HDD 면 ↓
```

### 3.1 force_merge — 운영 시 수동

```http
POST /logs-2026-04/_forcemerge?max_num_segments=1
```

- 오래된 (변경 X) 인덱스에 적용 → 검색 ↑, 디스크 ↓
- **신중히** — I/O 폭증

---

## 4. 검색 성능

### 4.1 Filter Context 사용

점수 X + 캐시:

```json
{
  "bool": {
    "must":   [{ "match": { "title": "elastic" } }],
    "filter": [
      { "term": { "status": "active" } },
      { "range": { "createdAt": { "gte": "now-7d/d" } } }
    ]
  }
}
```

### 4.2 Now 캐시

`now-1d/d` — 일 단위로 round → 캐시 효과 ↑.
`now-1d` — 매 호출 다른 ms → 캐시 X.

### 4.3 routing

```http
POST /products/_doc?routing=user_123
{ "userId": "user_123", ... }

GET /products/_search?routing=user_123
{ "query": ... }
```

→ 한 shard 만 hit. 큰 효과.

### 4.4 source filtering

```json
{ "_source": ["name", "price"] }
```

큰 doc 일부만 필요 → 네트워크 / 직렬화 ↓.

### 4.5 stored_fields / docvalue_fields

```json
{ "docvalue_fields": ["@timestamp"] }
```

이미 docvalue 인 필드는 컬럼 저장에서 빠르게.

### 4.6 페이지네이션

```
from + size → 10000 한계
큰 페이지 → search_after / PIT
```

자세히 → [[query-dsl#9-sort--pagination]]

### 4.7 deep aggregation
큰 cardinality → composite agg.

### 4.8 preference

```http
GET /products/_search?preference=_local
```

같은 사용자는 같은 shard → 캐시 활용.

---

## 5. 캐시

### 5.1 Node Query Cache

filter context 결과 캐시. LRU. 노드 단위.

```yaml
indices.queries.cache.size: 10%
```

### 5.2 Shard Request Cache

`size: 0` (집계만) 요청 결과 캐시.

```http
GET /products/_search?request_cache=true
```

기본 활성 (size: 0 시).

### 5.3 Fielddata

text 필드 agg/sort 시 사용. 메모리 ↑↑. **keyword 사용** 으로 회피.

```yaml
indices.fielddata.cache.size: 20%
```

---

## 6. 디스크 / OS

### 6.1 SSD / NVMe — 필수
Lucene 의 random read.

### 6.2 vm.max_map_count

```bash
sysctl -w vm.max_map_count=262144
```

mmap 부족 시 색인 실패.

### 6.3 noatime

```
/var/lib/elasticsearch  ext4  defaults,noatime  0 0
```

### 6.4 ulimit

```
nofile 65535
memlock unlimited
```

---

## 7. 모니터링

### 7.1 stats

```http
GET /_nodes/stats
GET /_nodes/stats/jvm
GET /_nodes/stats/indices
GET /products/_stats
```

### 7.2 hot_threads

```http
GET /_nodes/hot_threads?threads=3&interval=500ms
```

CPU 가 어디에 쓰이는지.

### 7.3 thread pool

```http
GET /_cat/thread_pool?v&h=name,active,queue,rejected
```

`rejected` > 0 → 부하 초과. queue 늘림 또는 노드 추가.

### 7.4 Slow Log

```http
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "1s",
  "index.indexing.slowlog.threshold.index.warn": "10s"
}
```

`logs/<cluster>_index_search_slowlog.log` 확인.

---

## 8. Profile API

```http
GET /products/_search
{
  "profile": true,
  "query": { ... }
}
```

각 lucene 단계의 시간. **디버그용** — 운영엔 X.

---

## 9. Shard 크기 / 개수

- shard = 10~50 GB 권장
- 너무 작음 → 메타데이터 부담
- 너무 큼 → 이동 / 복구 비용 ↑
- 노드당 shard ≤ 600 권장 (대략)

---

## 10. 자주 보는 적신호

| 신호 | 의미 |
| --- | --- |
| `Disk watermark high` | 디스크 ↑ → 정리 / 노드 추가 |
| `Pending tasks > 100` | master 부하 |
| `Search rejected` | thread pool 한계 |
| `GC time > 10%` | heap 부족 / 큰 query |
| `Old GC frequent` | OOM 임박 |
| `Yellow / red` | replica / shard 문제 |

---

## 11. 일반 운영 권장

1. Heap 50%, ≤32GB.
2. SSD + noatime + vm.max_map_count.
3. 명시 mapping (keyword vs text).
4. Filter context 활용.
5. ILM + rollover.
6. snapshot 정기.
7. Kibana / Stack Monitoring 으로 추세.
8. shard 가이드 준수.

---

## 12. 함정

### 12.1 Heap > 32GB
compressed OOPS 풀림 → 성능 ↓.

### 12.2 fielddata 활성
text 필드 agg → 메모리 폭증. keyword 사용.

### 12.3 too many shards
green 이지만 무거움. 컨솔리데이션 / ILM.

### 12.4 force_merge live index
ongoing 색인 인덱스에 force_merge → 부담. 옛 인덱스만.

### 12.5 큰 from+size
10000 한계 + 비효율. search_after / PIT.

### 12.6 큰 응답
`_source` filter. 필요한 필드만.

### 12.7 dynamic mapping 폭증
필드 1000 한계. flattened / runtime.

### 12.8 refresh_interval 변경 후 유지
임시로 늘렸으면 색인 후 다시 1s.

---

## 13. 학습 자료

- **Tune for indexing speed** — elastic.co/guide
- **Tune for search speed**
- **Elasticsearch in Action** Ch. 10-11
- **Elastic Stack Monitoring**

---

## 14. 관련

- [[configuration]] — JVM / OS
- [[cluster-shards]] — shard 가이드
- [[query-dsl]] — 효율적 쿼리
- [[elasticsearch]] — ES hub
