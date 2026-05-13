---
title: "Elasticsearch Query DSL — bool / match / term / range"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:40:00+09:00
tags:
  - database
  - elasticsearch
  - query
---

# Elasticsearch Query DSL

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | bool / match / term / range / 검색 |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. Query Context vs Filter Context

| | Query | Filter |
| --- | --- | --- |
| 점수 (score) | 계산 ✅ | 계산 X |
| 캐시 | X | ✅ |
| 용도 | "얼마나 잘 맞나" | "맞나/안 맞나" |

```json
{
  "query": {
    "bool": {
      "must":   [ { "match": { "title": "elastic" } } ],   // 점수 ↑
      "filter": [ { "term":  { "status": "active" } } ]    // 점수 X, 캐시
    }
  }
}
```

→ 점수가 의미 없는 조건은 **filter** 로.

---

## 2. Full-Text — match 계열

### 2.1 match

```json
{ "query": { "match": { "title": "elastic search" } } }
```

`elastic search` → 토큰화 → `elastic` OR `search` 검색 (기본 OR).
operator AND:

```json
{ "match": { "title": { "query": "elastic search", "operator": "and" } } }
```

### 2.2 match_phrase

```json
{ "match_phrase": { "title": "elastic search" } }
```

토큰 순서 + 인접.

### 2.3 match_phrase_prefix

```json
{ "match_phrase_prefix": { "title": "elastic sea" } }
```

자동 완성 / search-as-you-type.

### 2.4 multi_match

```json
{
  "multi_match": {
    "query": "elastic",
    "fields": ["title^3", "body", "tags"],
    "type": "best_fields"
  }
}
```

| type | 의미 |
| --- | --- |
| `best_fields` | 가장 좋은 필드 점수 |
| `most_fields` | 모든 필드 점수 합 |
| `cross_fields` | 한 필드로 결합 |
| `phrase` | match_phrase 적용 |
| `phrase_prefix` | match_phrase_prefix |
| `bool_prefix` | match_bool_prefix |

### 2.5 query_string / simple_query_string

```json
{ "query_string": { "query": "(elastic OR mongo) AND search" } }
{ "simple_query_string": { "query": "+elastic -mongo \"full text\"" } }
```

---

## 3. Term-level (keyword / 정확)

### 3.1 term

```json
{ "term": { "status": "active" } }
```

⚠️ `text` 필드에 사용 시 분석 안 됨 → 매칭 안 될 수 있음. **keyword 필드에**.

### 3.2 terms

```json
{ "terms": { "tags": ["a","b","c"] } }
```

### 3.3 range

```json
{ "range": { "price": { "gte": 10, "lte": 100 } } }
{ "range": { "@timestamp": { "gte": "now-7d/d", "lt": "now/d" } } }
```

### 3.4 exists

```json
{ "exists": { "field": "email" } }
```

### 3.5 prefix / wildcard / regexp

```json
{ "prefix":   { "name": "ali" } }
{ "wildcard": { "name": "a*ce" } }
{ "regexp":   { "name": "a.*" } }
```

⚠️ wildcard / regexp 시작 와일드카드는 느림.

### 3.6 fuzzy

```json
{ "fuzzy": { "name": { "value": "alicia", "fuzziness": "AUTO" } } }
```

Levenshtein 거리.

### 3.7 ids

```json
{ "ids": { "values": ["1","2","3"] } }
```

---

## 4. Bool Query

```json
{
  "bool": {
    "must":     [ ... ],   // AND (점수)
    "should":   [ ... ],   // OR (점수)
    "must_not": [ ... ],   // NOT (점수 X)
    "filter":   [ ... ],   // AND (점수 X, 캐시)
    "minimum_should_match": 1
  }
}
```

### 4.1 패턴

```json
// "elastic" 단어 검색 + 활성 + 가격 범위
{
  "query": {
    "bool": {
      "must":   [{ "match": { "title": "elastic" } }],
      "filter": [
        { "term":  { "status": "active" } },
        { "range": { "price": { "gte": 10, "lte": 100 } } }
      ]
    }
  }
}
```

---

## 5. Compound

### 5.1 dis_max — 가장 좋은 점수

```json
{ "dis_max": {
    "queries": [
      { "match": { "title": "elastic" } },
      { "match": { "body":  "elastic" } }
    ],
    "tie_breaker": 0.3
  }
}
```

### 5.2 function_score

```json
{ "function_score": {
    "query": { "match": { "title": "elastic" } },
    "functions": [
      { "filter": { "term": { "featured": true } }, "weight": 5 },
      { "field_value_factor": { "field": "popularity", "factor": 1.2 } }
    ],
    "score_mode": "sum",
    "boost_mode": "multiply"
  }
}
```

### 5.3 boosting

```json
{ "boosting": {
    "positive": { "match": { "title": "elastic" } },
    "negative": { "match": { "title": "outdated" } },
    "negative_boost": 0.2
  }
}
```

### 5.4 constant_score

```json
{ "constant_score": { "filter": { "term": { "status": "active" } } } }
```

---

## 6. Nested / Parent-Child

### 6.1 nested

```json
{ "nested": {
    "path": "comments",
    "query": {
      "bool": {
        "must": [
          { "match": { "comments.text": "great" } },
          { "term":  { "comments.author": "alice" } }
        ]
      }
    }
  }
}
```

### 6.2 has_child / has_parent (join 필드)

```json
{ "has_child": { "type": "comment", "query": { "match": { "text": "great" } } } }
```

---

## 7. Geo

```json
{ "geo_distance": {
    "distance": "1km",
    "location": { "lat": 37.5, "lon": 127.0 }
  }
}

{ "geo_bounding_box": {
    "location": {
      "top_left":     { "lat": 38.0, "lon": 126.5 },
      "bottom_right": { "lat": 37.0, "lon": 127.5 }
    }
  }
}

{ "geo_shape": {
    "area": {
      "shape": { "type": "Polygon", "coordinates": [...] },
      "relation": "within"
    }
  }
}
```

---

## 8. KNN / Vector (8.0+)

```json
GET /products/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": { "term": { "category": "shoes" } }
  }
}

// Hybrid (KNN + 텍스트)
{
  "knn": { ... },
  "query": { "match": { "title": "running" } },
  "rank": { "rrf": {} }
}
```

---

## 9. Sort / Pagination

```json
{
  "from": 0,
  "size": 10,
  "sort": [
    { "createdAt": "desc" },
    "_score",
    { "name.raw": "asc" }
  ]
}
```

### 9.1 큰 페이지 — search_after

```json
{
  "sort": [{ "createdAt": "desc" }, { "_id": "asc" }],
  "search_after": [1700000000, "doc_id_xxx"]
}
```

→ from/size 의 한계 (10000) 회피.

### 9.2 PIT (Point In Time)

```http
POST /products/_pit?keep_alive=1m
# → { "id": "..." }

GET /_search
{
  "pit": { "id": "...", "keep_alive": "1m" },
  "size": 100,
  "sort": [...]
}
```

scroll 의 후계 (7.10+).

---

## 10. 부분 응답 — Source Filtering

```json
{
  "_source": ["name", "price"],
  "_source": { "includes": ["*.name"], "excludes": ["password"] }
}
```

### 10.1 fields / docvalue_fields / script_fields

```json
{
  "fields": ["name", "price"],
  "docvalue_fields": ["@timestamp"],
  "script_fields": {
    "total_with_tax": {
      "script": "doc['price'].value * 1.1"
    }
  }
}
```

---

## 11. Highlighting

```json
{
  "query": { "match": { "body": "elastic" } },
  "highlight": {
    "fields": {
      "body": { "pre_tags": ["<em>"], "post_tags": ["</em>"] }
    }
  }
}
```

---

## 12. Aggregation 과 함께

```json
{
  "query": { "match": { "category": "electronics" } },
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "by_brand":  { "terms": { "field": "brand" } }
  }
}
```

자세히 → [[aggregations]]

---

## 13. Multi-Search / msearch

```http
POST /_msearch
{}
{ "query": { "match_all": {} } }
{ "index": "logs" }
{ "query": { "match": { "level": "error" } } }
```

여러 검색 한 번에.

---

## 14. ESQL (8.11+, OpenSearch SQL)

```http
POST /_query
{ "query": "FROM products | WHERE price > 10 | STATS avg(price) BY category" }
```

SQL-like pipeline.

---

## 15. 함정

### 15.1 `term` on `text` 필드
text 는 분석 → token 매칭 안 됨. keyword 필드에.

### 15.2 `must` vs `filter` 혼동
점수 필요 없으면 filter. 캐시 효과.

### 15.3 from+size > 10000
deep pagination — search_after / PIT.

### 15.4 `match_all` 으로 큰 데이터 조회
PIT + search_after 또는 scroll (옛).

### 15.5 `_score` 안 보고 sort
sort 추가 시 _score 미계산. `track_scores: true`.

### 15.6 wildcard 시작
`*foo*` 풀스캔. wildcard 타입 또는 ngram 인덱스.

### 15.7 `query_string` 파싱 에러
사용자 입력 그대로 → 에러. simple_query_string 권장.

### 15.8 nested 안의 sort
`nested_path` 명시 필요.

---

## 16. 학습 자료

- **Query DSL Reference**
- **Elasticsearch in Action** Ch. 5-7
- **Relevance Tuning** — elastic.co/blog

---

## 17. 관련

- [[index-mapping]] — text vs keyword
- [[aggregations]] — agg 결합
- [[elasticsearch]] — ES hub
