---
title: "Elasticsearch 통합 (한국어 nori / 색인 동기화)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - elasticsearch
  - search
---

# Elasticsearch 통합 (한국어 nori / 색인 동기화)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | mapping / nori 분석기 / outbox 색인 / 검색 query |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]]. PG → ES 전환 시점: [[product-search#6]].

---

## 1. 언제 ES 로

| 신호 | PG 한계 |
| --- | --- |
| `LIKE '%xxx%'` 가 느림 (1초+) | trgm GIN 으로 못 메우는 분량 |
| 오타 / 동의어 / 한국어 분석 | PG `simple` 토크나이저 부족 |
| 가중 랭킹 (boost / freshness) | 표현 어려움 |
| 자동완성 / 인기 검색어 | suggester 없음 |
| 100만+ 상품 | trgm 도 한계 |

→ 위 중 2개 이상 = ES 도입.

---

## 2. 구성

```
PG (진실의 원천) ──→ outbox 또는 CDC (Debezium) ──→ Kafka ──→ ES 색인 워커 ──→ Elasticsearch
                                                                        ↑
                                                                 GET /api/search
                                                                        │
                                                                  ProductSearchService
                                                                  (ES query)
```

---

## 3. 의존성

```kotlin
implementation("co.elastic.clients:elasticsearch-java:8.13.4")
// 또는 Spring Data Elasticsearch (간단하지만 8.x 호환 주의)
implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
```

```yaml
spring:
  elasticsearch:
    uris: ${ES_URIS:http://localhost:9200}
    username: ${ES_USER:}
    password: ${ES_PASSWORD:}
    socket-timeout: 5s
    connection-timeout: 3s

app:
  search:
    index:
      product: products_v1                     # 버전 prefix
```

---

## 4. Index Mapping — 한국어 nori 분석기

```json
PUT products_v1
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "korean_nori": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["lowercase", "nori_part_of_speech", "stemmer"]
        },
        "korean_nori_search": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["lowercase"]
        },
        "autocomplete": {
          "type": "custom",
          "tokenizer": "korean_nori",
          "filter": ["lowercase", "edge_ngram_filter"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id":           { "type": "keyword" },
      "name":         {
        "type": "text",
        "analyzer": "korean_nori",
        "search_analyzer": "korean_nori_search",
        "fields": {
          "keyword": { "type": "keyword" },
          "auto":    { "type": "text", "analyzer": "autocomplete", "search_analyzer": "korean_nori_search" }
        }
      },
      "brand": {
        "properties": {
          "id":   { "type": "keyword" },
          "name": { "type": "text", "analyzer": "korean_nori" }
        }
      },
      "categoryPath":  { "type": "keyword" },          // "/men/top/hoodie"
      "tags":          { "type": "keyword" },
      "basePriceKrw":  { "type": "long" },
      "saleCount30d":  { "type": "long" },
      "reviewAvg":     { "type": "float" },
      "reviewCount":   { "type": "integer" },
      "status":        { "type": "keyword" },
      "createdAt":     { "type": "date" },
      "updatedAt":     { "type": "date" }
    }
  }
}
```

**nori plugin** — Elastic 의 한국어 분석기. 설치:

```bash
bin/elasticsearch-plugin install analysis-nori
```

---

## 5. 색인 동기화 — 3 방법

### 5.1 Outbox 패턴 (권장)

```sql
CREATE TABLE search_index_outbox (
    id            CHAR(26) PRIMARY KEY,
    target_type   VARCHAR(30) NOT NULL,            -- PRODUCT / USER
    target_id     CHAR(26) NOT NULL,
    operation     VARCHAR(20) NOT NULL,             -- UPSERT / DELETE
    payload       JSONB,
    status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts      INTEGER NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at  TIMESTAMPTZ
);
CREATE INDEX ix_search_outbox_pending ON search_index_outbox (status, created_at)
    WHERE status = 'PENDING';
```

```java
@Service
@RequiredArgsConstructor
public class SearchIndexOutboxService {

    private final SearchIndexOutboxRepository outbox;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional(propagation = Propagation.MANDATORY)
    public void enqueue(String targetType, String targetId, String operation, Object payload) {
        outbox.save(new SearchIndexOutboxRow(
            ids.next(), targetType, targetId, operation,
            payload, "PENDING", 0, Instant.now(clock), null
        ));
    }
}

// 도메인 이벤트 listener
@Component
@RequiredArgsConstructor
public class ProductIndexEnqueuer {

    private final SearchIndexOutboxService outbox;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void on(ProductCreated event) {
        outbox.enqueue("PRODUCT", event.productId().value(), "UPSERT", null);
    }
    // ProductUpdated / ProductActivated / ProductDeleted 동일
}
```

```java
// Worker
@Component
@RequiredArgsConstructor
public class SearchIndexWorker {

    private final SearchIndexOutboxRepository outbox;
    private final ElasticsearchClient es;
    private final ProductRepository products;

    @Scheduled(fixedDelay = 1000)
    @SchedulerLock(name = "searchIndexWorker", lockAtMostFor = "5m")
    public void process() {
        var batch = outbox.findPending(100);
        for (var row : batch) processOne(row.id());
    }

    @Transactional
    public void processOne(SearchIndexOutboxId id) {
        var row = outbox.findByIdForUpdate(id).orElseThrow();
        if (!"PENDING".equals(row.status())) return;

        try {
            switch (row.operation()) {
                case "UPSERT" -> {
                    var product = products.findById(new ProductId(row.targetId()))
                        .orElseThrow(() -> new IllegalStateException("deleted"));
                    var doc = buildDoc(product);
                    es.index(i -> i.index("products_v1").id(product.id().value()).document(doc));
                }
                case "DELETE" -> es.delete(d -> d.index("products_v1").id(row.targetId()));
            }
            row.markProcessed(Instant.now());
        } catch (Exception e) {
            row.recordFailure(e.getMessage());
        }
        outbox.save(row);
    }
}
```

### 5.2 Debezium CDC

PG WAL 의 변경을 Kafka 로 → Kafka Connect ES sink. 코드 변경 0, 인프라 부담 ↑.

### 5.3 동기 (간단 — 작은 서비스)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void on(ProductUpdated event) {
    es.index(...);              // ES 직접 호출
}
```

→ ES 다운 시 색인 누락. **outbox 권장**.

---

## 6. 검색 query

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchClient es;

    public SearchResult search(ProductSearchCriteria c, int limit, Cursor cursor) {
        var bool = QueryBuilders.bool();

        // status filter
        bool.filter(f -> f.term(t -> t.field("status").value("ACTIVE")));

        // 키워드 검색 (name + brand)
        if (c.q() != null && !c.q().isBlank()) {
            bool.must(m -> m.multiMatch(mm -> mm
                .query(c.q())
                .fields("name^3", "brand.name^2", "tags")
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            ));
        }

        // 브랜드 (다중)
        if (c.brand() != null && !c.brand().isEmpty()) {
            bool.filter(f -> f.terms(t -> t.field("brand.id")
                .terms(tv -> tv.value(c.brand().stream().map(FieldValue::of).toList()))));
        }

        // 카테고리 (path prefix)
        if (c.categoryPath() != null) {
            bool.filter(f -> f.prefix(p -> p.field("categoryPath").value(c.categoryPath())));
        }

        // 가격 range
        if (c.priceMin() != null || c.priceMax() != null) {
            bool.filter(f -> f.range(r -> {
                var rq = r.field("basePriceKrw");
                if (c.priceMin() != null) rq.gte(JsonData.of(c.priceMin()));
                if (c.priceMax() != null) rq.lte(JsonData.of(c.priceMax()));
                return rq;
            }));
        }

        var sort = switch (c.sort()) {
            case LATEST -> SortOptions.of(s -> s.field(f -> f.field("createdAt").order(SortOrder.Desc)));
            case PRICE_ASC -> SortOptions.of(s -> s.field(f -> f.field("basePriceKrw").order(SortOrder.Asc)));
            case POPULAR -> SortOptions.of(s -> s.field(f -> f.field("saleCount30d").order(SortOrder.Desc)));
            default -> SortOptions.of(s -> s.score(sc -> sc.order(SortOrder.Desc)));
        };

        try {
            var response = es.search(s -> s
                .index("products_v1")
                .query(bool.build()._toQuery())
                .sort(sort)
                .size(limit)
                .searchAfter(cursor == null ? null : List.of(FieldValue.of(cursor.lastSort()))),
                ProductDoc.class
            );
            return mapToResult(response);
        } catch (Exception e) {
            throw new BusinessException(ResponseCode.EXTERNAL_API_ERROR, "검색 실패: " + e.getMessage());
        }
    }
}
```

---

## 7. 자동완성

```java
public List<String> autocomplete(String prefix, int limit) {
    var response = es.search(s -> s
        .index("products_v1")
        .query(q -> q.match(m -> m.field("name.auto").query(prefix)))
        .size(limit)
        .source(src -> src.filter(f -> f.includes("name"))),
        ProductDoc.class
    );
    return response.hits().hits().stream()
        .map(h -> h.source().name())
        .distinct()
        .toList();
}
```

→ `edge_ngram` 토큰 + 매칭으로 prefix 검색. 동의어 사전 + popular query log 활용 가능.

---

## 8. 인덱스 마이그레이션 (zero-downtime)

```
products_v1 → products_v2 (새 mapping)
```

1. `products_v2` 새 인덱스 생성 (새 mapping)
2. Reindex API 로 `v1 → v2` 복사
3. 진행 중 변경분은 outbox 가 v1, v2 둘 다 인덱스
4. 완료 후 alias `products` 를 v2 로 swap
5. v1 삭제

```bash
POST _reindex
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" }
}

POST _aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
```

→ 코드는 항상 alias `products` 사용.

---

## 9. 함정 모음

### 함정 1 — 동기 색인
ES 다운 시 비즈니스 트랜잭션 실패. **outbox + worker**.

### 함정 2 — nori plugin 미설치
한국어 분석 X — "후드"가 토큰화 안 됨. **plugin install** + analyzer 설정.

### 함정 3 — index mapping 변경 안 됨
`type` / `analyzer` 변경 = reindex 필요. **새 index + alias swap**.

### 함정 4 — `_source` 전체 반환
큰 document = 응답 느림. `source includes` 로 필요한 필드만.

### 함정 5 — deep pagination (`from + size`)
10000+ offset 시 ES 가 거부. **`search_after`** (cursor) 사용.

### 함정 6 — refresh 직후 즉시 검색 기대
기본 refresh interval 1초. **`?refresh=wait_for`** 또는 1초 대기.

### 함정 7 — 색인 worker 가 retry 무한
실패 횟수 limit + dead letter.

### 함정 8 — ES 단일 노드 운영
SPOF. 최소 3 노드 (마스터+데이터) cluster.

### 함정 9 — boost 의 explosion
`^100` boost 가 다른 점수를 압도. 0.5~3 범위.

### 함정 10 — outbox 처리 순서
같은 product 의 UPSERT 가 여러 번 enqueue → 처리 순서 보장 (`partition by targetId` 또는 단일 worker).

---

## 10. 운영 체크리스트

- [ ] nori plugin 설치
- [ ] outbox 패턴 + 별도 worker
- [ ] alias 로 index 추상화 (zero-downtime)
- [ ] `search_after` cursor pagination
- [ ] 색인 size / 인덱싱 throughput 모니터
- [ ] 검색 latency p95
- [ ] cluster 최소 3 노드
- [ ] snapshot 정기 (S3)
- [ ] index lifecycle policy (오래된 로그 hot/warm/cold/delete)

---

## 11. 관련

- [[product-search]] — PG 검색 (전 단계)
- [[webhook-send]] — outbox 패턴 동일
- [[../common/response-envelope]]
- [[api-design|↑ api-design hub]]
