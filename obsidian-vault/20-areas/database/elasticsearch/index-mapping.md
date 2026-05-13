---
title: "Elasticsearch Mapping — Field Type / Dynamic / Template"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:35:00+09:00
tags:
  - database
  - elasticsearch
  - mapping
---

# Elasticsearch Mapping

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 타입 / dynamic / template |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. Mapping 한 줄

각 필드의 **타입 + 색인 방식** 을 정의. 잘못 정의된 mapping = 검색 실패 / 비효율.

---

## 2. 기본 타입

### 2.1 텍스트

| 타입 | 의미 |
| --- | --- |
| **`text`** | 분석 (tokenize) — 검색용 |
| **`keyword`** | 그대로 — 필터 / 집계 / 정렬 |
| `wildcard` | 큰 텍스트 + 와일드카드 |
| `match_only_text` | text 의 가벼운 버전 (8.0+) |
| `search_as_you_type` | 자동 완성 |

```json
"name": { "type": "text" }
"status": { "type": "keyword" }

// 둘 다 (multi-field)
"name": {
  "type": "text",
  "fields": {
    "raw": { "type": "keyword" }
  }
}
// 검색: name, 필터/정렬: name.raw
```

### 2.2 숫자

| 타입 | 의미 |
| --- | --- |
| `long`, `integer`, `short`, `byte` | 정수 |
| `double`, `float`, `half_float` | 부동소수 |
| `scaled_float` | 정수 × scale |
| `unsigned_long` | 0 이상 |

```json
"price": { "type": "double" }
"score": { "type": "scaled_float", "scaling_factor": 100 }
```

### 2.3 날짜

```json
"createdAt": {
  "type": "date",
  "format": "strict_date_optional_time||epoch_millis"
}

"date_nanos"   // ns 정밀도 (8.x+)
```

### 2.4 불리언

```json
"active": { "type": "boolean" }
```

### 2.5 객체 / 중첩

```json
"address": {
  "type": "object",
  "properties": {
    "city": { "type": "keyword" },
    "zip":  { "type": "keyword" }
  }
}

// nested — 배열 안 객체의 독립성 유지
"comments": {
  "type": "nested",
  "properties": { ... }
}
```

⚠️ `object` 의 배열은 내부적으로 "flatten" → 객체 경계 사라짐. **nested** 가 진짜 객체 보존.

### 2.6 IP / 지리 / 벡터

```json
"client_ip": { "type": "ip" }

"location": { "type": "geo_point" }
"area":     { "type": "geo_shape" }

"embedding": { "type": "dense_vector", "dims": 1536, "index": true, "similarity": "cosine" }
"sparse_vec": { "type": "sparse_vector" }
```

### 2.7 다른

- `binary` — base64
- `range` (integer_range, date_range, ip_range)
- `flattened` — 알 수 없는 키 (한 필드로 flatten)
- `join` — 부모-자식
- `alias` — 다른 필드의 별칭

---

## 3. 명시적 mapping

```http
PUT /products
{
  "mappings": {
    "properties": {
      "name":    { "type": "text", "analyzer": "standard" },
      "price":   { "type": "double" },
      "stock":   { "type": "integer" },
      "tags":    { "type": "keyword" },
      "createdAt": { "type": "date" }
    }
  }
}
```

### 3.1 추가만 가능
**필드 추가는 가능, 기존 필드 타입 변경은 X**. 변경 필요 → 새 인덱스 + reindex.

```http
PUT /products/_mapping
{
  "properties": {
    "description": { "type": "text" }
  }
}
```

---

## 4. Dynamic Mapping

자동 추정 — 첫 색인 시 추측.

```json
"timestamp": "2026-05-13"   → "date"
"name": "Alice"             → "text" + "keyword" 서브필드
"score": 42                 → "long"
"price": 1.5                → "float"
```

### 4.1 끄기 / 제어

```json
"mappings": {
  "dynamic": "strict",        // 정의 안 한 필드 X
  // "dynamic": false,        // 무시 (저장만, 인덱스 X)
  // "dynamic": "runtime",    // runtime field 로
  "properties": { ... }
}
```

### 4.2 Dynamic Template

```json
"mappings": {
  "dynamic_templates": [
    {
      "strings_as_keyword": {
        "match_mapping_type": "string",
        "mapping": { "type": "keyword", "ignore_above": 256 }
      }
    },
    {
      "long_to_double": {
        "match_mapping_type": "long",
        "mapping": { "type": "double" }
      }
    }
  ]
}
```

---

## 5. Multi-field

```json
"title": {
  "type": "text",
  "analyzer": "korean",
  "fields": {
    "raw":    { "type": "keyword" },
    "ngram":  { "type": "text", "analyzer": "ngram_analyzer" },
    "english":{ "type": "text", "analyzer": "english" }
  }
}
```

쿼리: `title`, `title.raw`, `title.ngram`, `title.english`.

---

## 6. 옵션

### 6.1 index — 검색 가능 여부

```json
"private": { "type": "keyword", "index": false }   // 저장만, 검색 X
```

### 6.2 doc_values — 정렬 / 집계

```json
"name": { "type": "text", "doc_values": false }    // text 는 기본 X
"category": { "type": "keyword" }                  // keyword 는 기본 ON
```

### 6.3 store — _source 와 별도

```json
"large": { "type": "text", "store": true }
```

기본 `_source` 에서 가져옴. store 는 필드 단위 저장 (특수 케이스).

### 6.4 ignore_above

```json
"url": { "type": "keyword", "ignore_above": 256 }
```

256 자 초과 → 인덱싱 X (저장만).

### 6.5 null_value

```json
"status": { "type": "keyword", "null_value": "UNKNOWN" }
```

### 6.6 copy_to

```json
"first": { "type": "text", "copy_to": "full_name" },
"last":  { "type": "text", "copy_to": "full_name" },
"full_name": { "type": "text" }
```

여러 필드를 한 검색 필드로.

---

## 7. Index Template (자동 mapping)

```http
PUT /_index_template/logs
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "index.lifecycle.name": "logs-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" }
      }
    }
  },
  "priority": 100
}
```

새 `logs-*` 인덱스 생성 시 자동 적용.

### 7.1 Component Template (재사용)

```http
PUT /_component_template/common_fields
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" }
      }
    }
  }
}

PUT /_index_template/logs
{
  "index_patterns": ["logs-*"],
  "composed_of": ["common_fields"]
}
```

---

## 8. Runtime Field (7.11+)

색인 X — 쿼리 시 계산.

```json
"mappings": {
  "runtime": {
    "day_of_week": {
      "type": "keyword",
      "script": "emit(doc['@timestamp'].value.dayOfWeekEnum.toString())"
    }
  }
}
```

장점: schema 자유 / 마이그 없이 추가.
단점: 쿼리 시간 ↑.

---

## 9. Reindex (mapping 변경 시)

```http
POST /_reindex
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" }
}

// 비동기 (큰 데이터)
POST /_reindex?wait_for_completion=false
```

alias 로 무중단 전환:

```http
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
```

---

## 10. 함정

### 10.1 Dynamic mapping 의 자동 결정
첫 색인이 잘못 → 영구 잘못. **명시 mapping** 권장.

### 10.2 text vs keyword 혼동
text 로 정렬 / 집계 → X 또는 fielddata 활성 (메모리 ↑). keyword 사용.

### 10.3 object 배열의 flatten
중첩 객체 배열은 nested 필요.

### 10.4 매핑 변경 시도
기존 필드 타입 변경 X. 새 인덱스 + reindex + alias.

### 10.5 너무 많은 필드
default 1000 필드 한계. flattened / runtime 검토.

### 10.6 너무 깊은 객체
default 20 depth. 평탄화.

### 10.7 dense_vector dims 변경
변경 X — reindex.

### 10.8 copy_to 와 source
`_source` 에는 원본만. copy_to 는 인덱스 전용.

---

## 11. 학습 자료

- **Mapping Reference** — elastic.co/guide → mapping
- **Mapping Best Practices** — elastic.co/blog
- **Field datatypes**

---

## 12. 관련

- [[analyzer-korean]] — text 분석
- [[query-dsl]] — text vs keyword 쿼리
- [[elasticsearch]] — ES hub
