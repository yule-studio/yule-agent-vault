---
title: "Elasticsearch Aggregations — Metric / Bucket / Pipeline"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:45:00+09:00
tags:
  - database
  - elasticsearch
  - aggregation
---

# Elasticsearch Aggregations

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | metric / bucket / pipeline |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 한 줄

Aggregations = ES 의 **GROUP BY + 통계**. 분석 / 대시보드 / facet 의 핵심.
3 카테고리: **Metric**, **Bucket**, **Pipeline**.

---

## 2. 기본 구조

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "total_revenue": { "sum": { "field": "amount" } },
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

- `size: 0` — hits 안 받고 agg 만 (효율)
- agg 안에 agg 중첩 가능

---

## 3. Metric Aggregations

```json
{
  "stats":         { "stats":         { "field": "price" } },
  "extended":      { "extended_stats":{ "field": "price" } },
  "min":           { "min":           { "field": "price" } },
  "max":           { "max":           { "field": "price" } },
  "sum":           { "sum":           { "field": "price" } },
  "avg":           { "avg":           { "field": "price" } },
  "median":        { "percentiles":   { "field": "price", "percents": [50] } },
  "p95":           { "percentiles":   { "field": "latency", "percents": [50,95,99] } },
  "p_rank":        { "percentile_ranks": { "field": "price", "values": [10, 50, 100] } },
  "unique_users":  { "cardinality":   { "field": "user_id", "precision_threshold": 1000 } },
  "value_count":   { "value_count":   { "field": "price" } },
  "top":           { "top_hits":      { "size": 3, "sort": [{ "score": "desc" }] } },
  "matrix":        { "matrix_stats":  { "fields": ["price", "stock"] } }
}
```

### 3.1 cardinality — HyperLogLog

대략 unique 수. 정확 X — `precision_threshold` (메모리 vs 정확).

### 3.2 percentile — T-Digest

p50/p95/p99 등. 정확 X — 대용량에 효율.

### 3.3 top_hits

각 버킷의 대표 문서:

```json
{
  "by_user": {
    "terms": { "field": "userId" },
    "aggs": {
      "latest": {
        "top_hits": {
          "size": 1,
          "sort": [{ "createdAt": "desc" }]
        }
      }
    }
  }
}
```

---

## 4. Bucket Aggregations

### 4.1 terms — GROUP BY

```json
{ "by_category": {
    "terms": {
      "field": "category",
      "size": 10,
      "order": { "_count": "desc" },
      "missing": "N/A"
    }
  }
}
```

`field` 는 keyword (또는 numeric). text 는 안 됨 (fielddata 활성 필요).

### 4.2 multi-terms (composite)

```json
{ "by_brand_cat": {
    "multi_terms": {
      "terms": [
        { "field": "brand" },
        { "field": "category" }
      ]
    }
  }
}
```

### 4.3 date_histogram

```json
{ "by_day": {
    "date_histogram": {
      "field": "@timestamp",
      "calendar_interval": "1d",
      "time_zone": "Asia/Seoul",
      "format": "yyyy-MM-dd",
      "min_doc_count": 0,
      "extended_bounds": { "min": "now-30d/d", "max": "now/d" }
    }
  }
}
```

| Interval | 종류 |
| --- | --- |
| `calendar_interval` | `1d`, `1w`, `1M` — 달력 |
| `fixed_interval` | `60s`, `3600s` — 고정 |

### 4.4 histogram

```json
{ "price_dist": {
    "histogram": { "field": "price", "interval": 10 }
  }
}
```

### 4.5 range / date_range

```json
{ "by_age": {
    "range": {
      "field": "age",
      "ranges": [
        { "to": 18, "key": "minor" },
        { "from": 18, "to": 65, "key": "adult" },
        { "from": 65, "key": "senior" }
      ]
    }
  }
}
```

### 4.6 filter / filters

```json
{ "active_only": {
    "filter": { "term": { "active": true } },
    "aggs": { "avg_price": { "avg": { "field": "price" } } }
  }
}

{ "by_status": {
    "filters": {
      "filters": {
        "active":   { "term": { "status": "active" } },
        "inactive": { "term": { "status": "inactive" } }
      }
    }
  }
}
```

### 4.7 nested

```json
{ "by_comment_author": {
    "nested": { "path": "comments" },
    "aggs": {
      "authors": { "terms": { "field": "comments.author" } }
    }
  }
}
```

### 4.8 geo_distance / geohash_grid

```json
{ "by_grid": {
    "geohash_grid": { "field": "location", "precision": 5 }
  }
}
```

### 4.9 sampler / diversified_sampler

큰 집계의 표본만 — 빠름.

### 4.10 composite — 페이지네이션 가능 agg

```json
{ "by_group": {
    "composite": {
      "size": 100,
      "sources": [
        { "brand": { "terms": { "field": "brand" } } },
        { "day":   { "date_histogram": { "field": "@timestamp", "calendar_interval": "1d" } } }
      ],
      "after": { "brand": "lg", "day": 1700000000 }
    }
  }
}
```

terms 의 큰 집계 (수만 버킷) 의 대안.

---

## 5. Pipeline Aggregations

다른 agg 의 결과를 입력으로.

### 5.1 cumulative_sum

```json
{ "by_day": {
    "date_histogram": { "field": "@timestamp", "calendar_interval": "1d" },
    "aggs": {
      "day_sales": { "sum": { "field": "amount" } },
      "cumsum":    { "cumulative_sum": { "buckets_path": "day_sales" } }
    }
  }
}
```

### 5.2 derivative / moving_avg / moving_fn

```json
{ "moving": {
    "moving_fn": {
      "buckets_path": "day_sales",
      "window": 7,
      "script": "MovingFunctions.unweightedAvg(values)"
    }
  }
}
```

### 5.3 bucket_script

```json
{ "ratio": {
    "bucket_script": {
      "buckets_path": { "s": "day_sales", "c": "day_count" },
      "script": "params.s / params.c"
    }
  }
}
```

### 5.4 bucket_selector — 필터

```json
{ "filter_high": {
    "bucket_selector": {
      "buckets_path": { "avg": "avg_price" },
      "script": "params.avg > 100"
    }
  }
}
```

### 5.5 max_bucket / min_bucket / avg_bucket / stats_bucket

```json
{ "best_day": {
    "max_bucket": { "buckets_path": "by_day>day_sales" }
  }
}
```

→ "가장 매출 좋은 날 찾기".

---

## 6. agg + 정렬 (Top-N per group)

```json
{ "by_user": {
    "terms": {
      "field": "user_id",
      "size": 10,
      "order": { "total": "desc" }
    },
    "aggs": {
      "total":  { "sum": { "field": "amount" } },
      "recent": { "top_hits": { "size": 3, "sort": [{ "@timestamp": "desc" }] } }
    }
  }
}
```

---

## 7. Significant Terms / Significant Text

배경 빈도 대비 유의한 단어:

```json
{ "topic": {
    "significant_terms": { "field": "tag" }
  }
}
```

추천 / 트렌드 발견.

---

## 8. Cardinality 큰 데이터

```json
{ "by_user": {
    "terms": { "field": "userId", "size": 10000 }
  }
}
```

→ 비싸다. **composite** 또는 transform 으로 사전 계산.

---

## 9. Transform (8.x)

```http
PUT _transform/daily_summary
{
  "source": { "index": "events" },
  "pivot": {
    "group_by": {
      "date": { "date_histogram": { "field": "@timestamp", "calendar_interval": "1d" } },
      "user": { "terms": { "field": "userId" } }
    },
    "aggregations": {
      "events": { "value_count": { "field": "type" } }
    }
  },
  "dest": { "index": "daily_summary" }
}

POST _transform/daily_summary/_start
```

→ Materialized view. 매번 agg 재계산 안 함.

---

## 10. 함정

### 10.1 `terms` 의 정확도
shard 별 top N + 글로벌 top N → 일부 부정확. `shard_size` 늘리거나 composite.

### 10.2 큰 cardinality terms
수만 버킷 = 메모리 폭증. composite.

### 10.3 text 필드 agg
fielddata 활성 필요 (메모리 ↑). keyword 필드 / multi-field.

### 10.4 nested 없이 nested 필드 agg
잘못된 결과. nested agg 필수.

### 10.5 `size: 10000` (모든 버킷 받기)
큰 응답. composite + scroll.

### 10.6 percentile / cardinality 정확
HLL / T-Digest — 근사. 정확 필요하면 별도.

### 10.7 date_histogram timezone
서버 UTC 기본. `time_zone` 명시.

---

## 11. 학습 자료

- **Aggregations Reference**
- **Elasticsearch in Action** Ch. 9
- **Kibana visualizations** — agg 시각화

---

## 12. 관련

- [[query-dsl]] — query + agg
- [[index-mapping]] — keyword for agg
- [[elasticsearch]] — ES hub
