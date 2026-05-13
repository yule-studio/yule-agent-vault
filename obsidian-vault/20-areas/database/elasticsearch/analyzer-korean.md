---
title: "Elasticsearch Analyzer — 한국어 nori / 영어 / 사용자 정의"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:50:00+09:00
tags:
  - database
  - elasticsearch
  - analyzer
  - korean
---

# Elasticsearch Analyzer / 한국어 검색

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | analyzer / nori / synonym |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. Analyzer 구조

```
원본 텍스트
   ↓
Char Filter  (html_strip, mapping, ...)
   ↓
Tokenizer    (standard, whitespace, nori_tokenizer, ngram, ...)
   ↓
Token Filter (lowercase, stop, synonym, stemmer, ...)
   ↓
색인된 토큰
```

쿼리도 같은 analyzer 거침 → 검색 매칭.

---

## 2. 분석기 테스트

```http
POST /_analyze
{
  "analyzer": "standard",
  "text": "Elasticsearch is fast"
}

POST /_analyze
{
  "tokenizer": "nori_tokenizer",
  "text": "한국어 검색"
}

POST /my_index/_analyze
{
  "field": "title",
  "text": "..."
}
```

---

## 3. 내장 Analyzer

| Analyzer | 의미 |
| --- | --- |
| `standard` | 기본. Unicode 기반 |
| `simple` | lowercase + 비-문자로 분리 |
| `whitespace` | 공백만 |
| `stop` | + 불용어 |
| `keyword` | 분석 X (keyword 같음) |
| `pattern` | 정규식 |
| `language` (english, korean, ...) | 언어별 |

### 3.1 standard

```
"Quick Brown Fox 2026" → ["quick","brown","fox","2026"]
```

---

## 4. 한국어 — nori

### 4.1 설치

```bash
elasticsearch-plugin install analysis-nori
```

Elastic Cloud / OpenSearch 매니지드는 기본 포함.

### 4.2 사용

```http
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": [
            "nori_part_of_speech",
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "korean" }
    }
  }
}
```

### 4.3 nori_tokenizer 옵션

```json
"tokenizer": {
  "nori_user_dict": {
    "type": "nori_tokenizer",
    "decompound_mode": "mixed",       // none / discard / mixed
    "user_dictionary": "userdict_ko.txt",
    "user_dictionary_rules": ["커스텀단어"]
  }
}
```

| decompound_mode | 의미 |
| --- | --- |
| `none` | 복합어 분해 X |
| `discard` (기본) | 분해된 것만 |
| `mixed` | 원어 + 분해 |

예: `검색엔진` → mixed → `검색엔진`, `검색`, `엔진`.

### 4.4 nori_part_of_speech 필터

```json
"filter": {
  "ko_pos_filter": {
    "type": "nori_part_of_speech",
    "stoptags": ["E","J","SC","SE","SF","VCN","VCP","VV","VX"]
  }
}
```

조사 / 어미 등 제거.

### 4.5 nori_readingform

```
韓國 → 한국 (한자 → 한글 읽기)
```

### 4.6 사용자 사전

```
userdict_ko.txt:
  카드결제
  배달의민족
  당근마켓
```

---

## 5. 영어

```json
"english": {
  "type": "english"
}

// 또는 커스텀
"my_english": {
  "type": "custom",
  "tokenizer": "standard",
  "filter": [
    "lowercase",
    "english_stop",
    "english_keywords",
    "english_stemmer"
  ]
}
```

---

## 6. 다국어 — multi-field

```json
"title": {
  "type": "text",
  "analyzer": "korean",
  "fields": {
    "en": { "type": "text", "analyzer": "english" },
    "raw": { "type": "keyword" }
  }
}
```

쿼리:
```json
{
  "multi_match": {
    "query": "elastic",
    "fields": ["title", "title.en"]
  }
}
```

---

## 7. Token Filter

### 7.1 lowercase / uppercase

### 7.2 stop — 불용어

```json
"stop_filter": { "type": "stop", "stopwords": ["a","an","the"] }
```

### 7.3 synonym

```json
"my_synonyms": {
  "type": "synonym_graph",
  "synonyms": [
    "노트북, laptop, 랩탑",
    "스마트폰, smartphone => 휴대전화"
  ]
}
```

→ `노트북` 검색 시 `laptop`, `랩탑` 도 매칭.

⚠️ `index-time` vs `search-time` synonym — search-time 권장 (수정 쉬움).

### 7.4 ngram / edge_ngram

```json
"edge_ngram_filter": {
  "type": "edge_ngram",
  "min_gram": 2,
  "max_gram": 10
}
```

→ "elastic" → ["el", "ela", "elas", "elast", ...].
자동 완성 / 부분 검색.

### 7.5 shingle — 연속 토큰

```json
"shingle_filter": { "type": "shingle", "min_shingle_size": 2, "max_shingle_size": 3 }
```

→ "fast brown fox" → ["fast brown", "brown fox", "fast brown fox"].

### 7.6 stemmer

```json
"english_stemmer": { "type": "stemmer", "language": "english" }
// running → run
```

### 7.7 ascii_folding

```json
"asciifolding": { "type": "asciifolding", "preserve_original": true }
// café → cafe
```

### 7.8 word_delimiter_graph

```
"PowerShot-X100" → ["Power", "Shot", "X", "100", "PowerShot", "X100"]
```

---

## 8. Char Filter

### 8.1 html_strip

```json
"my_analyzer": {
  "char_filter": ["html_strip"],
  "tokenizer": "standard"
}
```

### 8.2 mapping

```json
"emoji_filter": {
  "type": "mapping",
  "mappings": [":) => happy", ":( => sad"]
}
```

### 8.3 pattern_replace

```json
{
  "type": "pattern_replace",
  "pattern": "(\\d+)",
  "replacement": "NUM"
}
```

---

## 9. 자동 완성 패턴

### 9.1 edge_ngram

```json
"analysis": {
  "analyzer": {
    "autocomplete": {
      "tokenizer": "autocomplete_tokenizer",
      "filter": ["lowercase"]
    },
    "autocomplete_search": {
      "tokenizer": "lowercase"
    }
  },
  "tokenizer": {
    "autocomplete_tokenizer": {
      "type": "edge_ngram",
      "min_gram": 2,
      "max_gram": 10,
      "token_chars": ["letter", "digit"]
    }
  }
}

"name": {
  "type": "text",
  "analyzer": "autocomplete",
  "search_analyzer": "autocomplete_search"
}
```

### 9.2 search_as_you_type 타입

```json
"name": { "type": "search_as_you_type" }
```

자동으로 ngram / shingle 인덱스 만듦.

### 9.3 completion suggester

```json
"name_sug": {
  "type": "completion",
  "analyzer": "korean"
}

// 검색
GET /products/_search
{
  "suggest": {
    "name_sug": {
      "prefix": "엘라",
      "completion": { "field": "name_sug" }
    }
  }
}
```

매우 빠른 prefix 검색.

---

## 10. analyzer 설정 변경

mapping 변경처럼 **새 인덱스 + reindex** 가 필요.
text 필드의 analyzer 는 색인 시 적용 → 기존 데이터 영향 X.

예외: `search_analyzer` 와 `synonyms` 만 검색 시 적용 → 동적 변경 가능 (settings update).

---

## 11. 한국어 + 영어 + 숫자 mix

```json
"korean_english": {
  "type": "custom",
  "tokenizer": "nori_tokenizer",
  "filter": [
    "lowercase",
    "nori_part_of_speech",
    "asciifolding",
    "english_stemmer"
  ]
}
```

영어 stemmer 가 한글 토큰엔 영향 없음.

---

## 12. 함정

### 12.1 인덱스 시점 / 쿼리 시점 analyzer 불일치
쿼리 단어가 인덱스와 다르게 분석 → 매칭 안 됨. 디버깅: `_analyze`.

### 12.2 nori 미설치
plugin 설치 필요. 또는 매니지드.

### 12.3 stopword 너무 많음
검색 빈도 ↓.

### 12.4 ngram 폭증
인덱스 크기 폭발. min/max 신중.

### 12.5 synonym index-time
변경 시 reindex 필요. search-time 권장.

### 12.6 text 필드만 분석
keyword 는 분석 X.

### 12.7 한국어 + standard
standard 는 한글을 음절로 안 자름 (CJK 토큰 다름). nori / icu / smart_cn 등 적합.

### 12.8 nori_part_of_speech 의 부작용
조사 모두 제거 → 의도와 다르게. 보수적으로 stoptags 정의.

---

## 13. 학습 자료

- **Text Analysis** — elastic.co/guide
- **Nori Plugin** — elastic.co/guide → analysis-nori
- **Elasticsearch in Action** Ch. 5

---

## 14. 관련

- [[index-mapping]] — text vs keyword
- [[query-dsl]] — match / synonym 활용
- [[elasticsearch]] — ES hub
