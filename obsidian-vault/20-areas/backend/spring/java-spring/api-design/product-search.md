---
title: "상품 검색·필터·정렬·페이지네이션 — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - commerce
  - search
  - pagination
---

# 상품 검색·필터·정렬·페이지네이션 — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Java 21 / cursor·offset / Specification·QueryDSL / N+1 회피 / Elasticsearch 단계 |

**[[api-design|↑ api-design hub]]**

> 전제: [[product-crud]]. `products`, `product_options`, `product_images`, `brands`, `categories`, `product_tags` 존재.
> **참고**: 무신사 (카테고리+브랜드+사이즈+가격대+인기), 쿠팡 (랭킹+로켓), 네이버 쇼핑 (자동완성+aggregator).

---

## 1. 무엇을 만드는가

```
GET /api/v1/products?q=후드&brand=01HZBRA...&category=01HZCAT...
                    &priceMin=20000&priceMax=80000
                    &size=M&size=L                           # 다중 옵션
                    &tag=오버사이즈
                    &sort=popular                            # popular | latest | priceAsc | priceDesc
                    &cursor=eyJ...&limit=20
```

### 1.1 요청 / 응답 — cursor 페이지네이션

```http
GET /api/v1/products?q=후드&sort=latest&limit=20

200 OK
{
  "data": {
    "items": [
      {
        "productId": "01HZ...",
        "name": "후드 차콜",
        "brand": { "id": "...", "name": "Acme" },
        "basePriceKrw": 49000,
        "mainImageUrl": "https://cdn.../1.jpg",
        "options": { "Size": ["S","M","L","XL"] },
        "soldOut": false,
        "createdAt": "2026-05-14T09:00:00Z"
      },
      ...
    ],
    "nextCursor": "eyJzIjoiTEFURVNUIiwidiI6IjIwMjYtMDUtMTRUMDk6MDA6MDBaIiwiaWQiOiIwMUhaLi4uIn0",
    "hasNext": true
  }
}
```

### 1.2 비기능

- p95 < **200 ms** 까지 PG 만 (그 이상 트래픽 / 자유 검색은 Elasticsearch)
- **cursor 페이지네이션** 기본 — 깊은 페이지 (50+) 에서 offset 의 성능 절벽 회피
- **offset 호환** 도 제공 — 관리자 / 일부 화면
- 필터 조합 = 2^k 인덱스 만들 수 없음 → **선택적 인덱스 + 통계 + Specification**
- DRAFT / DISCONTINUED / DELETED 는 제외
- N+1 회피 (옵션 / 이미지 / 브랜드)

---

## 2. cursor vs offset — 왜 cursor 가 기본인가

### 2.1 offset 의 함정

```sql
SELECT * FROM products WHERE status='ACTIVE'
ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

→ PG 는 **10020 행을 정렬해서 만들고 앞 10000개 버림**. OFFSET 큰 페이지 = O(offset). 50페이지부터 체감 시작.

### 2.2 cursor (keyset) 페이지네이션

```sql
SELECT * FROM products
WHERE status='ACTIVE'
  AND (created_at, id) < ('2026-05-14T09:00:00Z', '01HZ...')   -- 이전 페이지 마지막
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

→ 인덱스 `(created_at DESC, id DESC)` 만 있으면 **항상 O(limit)**. 1000페이지든 1만페이지든 같음.

### 2.3 사용 가이드

| 화면 | 권장 |
| --- | --- |
| 무한 스크롤 (모바일) | **cursor** |
| 페이지 번호 (1, 2, 3, ...) | offset (단, 깊이 제한) |
| 관리자 검색 (다양 sort) | offset + total count |
| 일반 상품 리스트 | cursor |

### 2.4 cursor 인코딩

```java
// 평문 cursor — 디버깅 쉬움, 조작 위험. 보통 base64 의 JSON.
record Cursor(SortKey sort, String lastValue, String lastId) {
    public String encode() {
        var json = """
            {"s":"%s","v":"%s","id":"%s"}
            """.formatted(sort, lastValue, lastId);
        return Base64.getUrlEncoder().withoutPadding()
            .encodeToString(json.getBytes(StandardCharsets.UTF_8));
    }
    public static Cursor decode(String raw) {
        try {
            var json = new String(Base64.getUrlDecoder().decode(raw), StandardCharsets.UTF_8);
            // Jackson 으로 파싱 권장
            return parseJson(json);
        } catch (Exception e) {
            throw new BadRequestException("invalid cursor");
        }
    }
}
```

> **함정**: cursor 안에 user 의 비즈니스 상태 (필터 / 정렬) 를 모두 담아두면 길어짐. **cursor = 마지막 항목 위치만**. 정렬 / 필터는 요청 파라미터로.

---

## 3. 필터 / 정렬 — 검색 전략 3단계

### 3.1 단계별 진화

| 단계 | 데이터 | 검색 도구 | 한계 |
| --- | --- | --- | --- |
| **1. PG 기본** | ~10만 상품 | `WHERE` + 인덱스 + Specification | 자유 텍스트 약함 |
| **2. PG + GIN** | ~100만 | `pg_trgm` / `tsvector` | 다국어 분석 약함 |
| **3. Elasticsearch** | 100만+ | nori / 다국어 / 자동완성 / 랭킹 | 인프라 + 색인 동기화 |

대부분의 스타트업은 **1번 → 3번** 으로 점프. 본 레시피는 **1번** 깊이.

### 3.2 인덱스 설계 (이미 product-crud 에 있음)

```sql
-- product_crud 에서 만든 인덱스
CREATE INDEX ix_products_brand_status     ON products (brand_id, status);
CREATE INDEX ix_products_category_status  ON products (category_id, status);
CREATE INDEX ix_products_updated_at       ON products (updated_at DESC);

-- 추가 — 가격대 필터 + 정렬 조합
CREATE INDEX ix_products_status_price     ON products (status, base_price_krw);
CREATE INDEX ix_products_status_created   ON products (status, created_at DESC, id DESC);
```

> **함정**: 인덱스 다 만들면 INSERT 가 느려짐. 실측해서 자주 쓰는 조합만. `pg_stat_user_indexes` 로 미사용 인덱스 추적.

### 3.3 자유 텍스트 — `pg_trgm` (단계 2)

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX ix_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- 검색
SELECT * FROM products WHERE name ILIKE '%후드%' AND status='ACTIVE';
-- pg_trgm 인덱스가 % 양쪽 wildcard 도 활용
```

### 3.4 정렬 (`sort=`)

| 값 | SQL |
| --- | --- |
| `latest` (기본) | `ORDER BY created_at DESC, id DESC` |
| `priceAsc` | `ORDER BY base_price_krw ASC, id ASC` |
| `priceDesc` | `ORDER BY base_price_krw DESC, id DESC` |
| `popular` | `ORDER BY sale_count_30d DESC, id DESC` (별도 집계 컬럼 필요) |

> **popular** 는 실시간 집계 X. **30일 판매 수** 같은 집계 컬럼 (배치 / 트리거 / Kafka consumer) 권장.

---

## 4. DB 보강 — 인기 / 검색용 비정규화

### 4.1 product_stats 테이블

```sql
-- V20__create_product_stats.sql — read 최적화용 비정규화
CREATE TABLE product_stats (
  product_id      CHAR(26) PRIMARY KEY REFERENCES products(id) ON DELETE CASCADE,
  sale_count_30d  BIGINT NOT NULL DEFAULT 0,
  view_count_30d  BIGINT NOT NULL DEFAULT 0,
  review_count    BIGINT NOT NULL DEFAULT 0,
  review_avg      NUMERIC(2,1) NOT NULL DEFAULT 0,        -- 0.0 ~ 5.0
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_product_stats_sale_30d ON product_stats (sale_count_30d DESC);
CREATE INDEX ix_product_stats_review_avg ON product_stats (review_avg DESC) WHERE review_count >= 5;
```

스케줄러 / Kafka consumer 가 갱신.

### 4.2 자주 쓰는 main image URL 도 비정규화

상품 목록 = 메인 이미지 URL 만 필요. 매번 `product_images` JOIN 은 비쌈.

```sql
ALTER TABLE products ADD COLUMN main_image_url VARCHAR(500);
-- 메인 이미지 추가 / 변경 시 트리거 또는 application 에서 동기화
```

---

## 5. 구현 — Spring Boot

### 5.1 요청 / 응답 DTO

```java
// presentation/api/v1/product/dto/ProductListRequest.java
public record ProductListRequest(
    String q,                                    // 검색어
    List<String> brand,                          // 다중 브랜드
    String category,                             // 단일 카테고리 (자식 포함)
    Long priceMin,
    Long priceMax,
    List<String> size,                           // Size 옵션 다중
    List<String> tag,
    SortKey sort,
    String cursor,
    @Min(1) @Max(100) int limit                  // 기본 20, 최대 100
) {
    public ProductListRequest {
        if (limit <= 0) limit = 20;
    }
}

public enum SortKey { LATEST, POPULAR, PRICE_ASC, PRICE_DESC }

public record ProductListResponse(
    List<ProductSummary> items,
    String nextCursor,
    boolean hasNext
) {
    public record ProductSummary(
        String productId,
        String name,
        BrandRef brand,
        long basePriceKrw,
        String mainImageUrl,
        Map<String, List<String>> options,     // {"Size":["S","M","L"]}
        boolean soldOut,
        long reviewCount,
        double reviewAvg,
        Instant createdAt
    ) {}
    public record BrandRef(String id, String name) {}
}
```

### 5.2 Specification 으로 동적 WHERE

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/product/ProductSpecs.java
public final class ProductSpecs {

    private ProductSpecs() {}

    public static Specification<ProductJpaEntity> activeOnly() {
        return (root, q, cb) -> cb.equal(root.get("status"), "ACTIVE");
    }

    public static Specification<ProductJpaEntity> nameContains(String text) {
        if (text == null || text.isBlank()) return null;
        var like = "%" + text.toLowerCase(Locale.ROOT).strip() + "%";
        return (root, q, cb) -> cb.like(cb.lower(root.get("name")), like);
    }

    public static Specification<ProductJpaEntity> brandIn(List<String> brandIds) {
        if (brandIds == null || brandIds.isEmpty()) return null;
        return (root, q, cb) -> root.get("brandId").in(brandIds);
    }

    public static Specification<ProductJpaEntity> categoryDescendantsOf(String path) {
        if (path == null) return null;
        // category_id 의 path 가 :path 로 시작 → 별도 조인. 여기선 단순화.
        return (root, q, cb) -> cb.like(root.get("categoryPath"), path + "%");
    }

    public static Specification<ProductJpaEntity> priceBetween(Long min, Long max) {
        return (root, q, cb) -> {
            var predicates = new ArrayList<Predicate>();
            if (min != null) predicates.add(cb.ge(root.get("basePriceKrw"), min));
            if (max != null) predicates.add(cb.le(root.get("basePriceKrw"), max));
            return predicates.isEmpty() ? null : cb.and(predicates.toArray(new Predicate[0]));
        };
    }

    public static Specification<ProductJpaEntity> hasAnyTag(List<String> tags) {
        if (tags == null || tags.isEmpty()) return null;
        return (root, q, cb) -> {
            var join = root.join("tags");           // @ElementCollection
            return join.in(tags);
        };
    }

    public static Specification<ProductJpaEntity> hasAnyOptionValue(String optionName, List<String> values) {
        if (values == null || values.isEmpty()) return null;
        return (root, q, cb) -> {
            var join = root.join("options");
            return cb.and(
                cb.equal(join.get("name"), optionName),
                join.get("value").in(values)
            );
        };
    }
}
```

### 5.3 cursor where 절

cursor 페이지네이션은 **정렬 키 + tie-breaker (id)** 의 조합 비교. Specification 만으론 어색하니 native query 또는 Criteria 직접.

```java
// CursorPredicate — 정렬별 (lastValue, lastId) 미만 조건
public static Specification<ProductJpaEntity> afterCursor(Cursor cursor, SortKey sort) {
    if (cursor == null) return null;
    return (root, q, cb) -> switch (sort) {
        case LATEST -> tupleLT(cb, root.get("createdAt"), root.get("id"),
                               Instant.parse(cursor.lastValue()), cursor.lastId());
        case PRICE_ASC -> tupleGT(cb, root.get("basePriceKrw"), root.get("id"),
                                  Long.parseLong(cursor.lastValue()), cursor.lastId());
        case PRICE_DESC -> tupleLT(cb, root.get("basePriceKrw"), root.get("id"),
                                   Long.parseLong(cursor.lastValue()), cursor.lastId());
        case POPULAR -> popularCursor(cb, root, cursor);
    };
}

// (a, b) < (av, bv) — JPA Criteria 에 직접 lexicographic 비교 없으니 OR 풀기
private static <V extends Comparable<V>> Predicate tupleLT(
        CriteriaBuilder cb, Path<V> a, Path<String> b, V av, String bv) {
    return cb.or(
        cb.lessThan(a, av),
        cb.and(cb.equal(a, av), cb.lessThan(b, bv))
    );
}
```

### 5.4 정렬 키 → JPA Sort 변환

```java
public static Sort toSort(SortKey key) {
    return switch (key == null ? SortKey.LATEST : key) {
        case LATEST     -> Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"));
        case PRICE_ASC  -> Sort.by(Sort.Order.asc("basePriceKrw"), Sort.Order.asc("id"));
        case PRICE_DESC -> Sort.by(Sort.Order.desc("basePriceKrw"), Sort.Order.desc("id"));
        case POPULAR    -> Sort.by(Sort.Order.desc("stats.saleCount30d"), Sort.Order.desc("id"));
    };
}
```

### 5.5 SearchProductsUseCase

```java
// src/main/java/com/example/shop/application/product/SearchProductsUseCase.java
@Service
public class SearchProductsUseCase {

    private final ProductJpaRepository repo;        // 검색은 직접 JPA repo 사용
    private final ProductSearchMapper mapper;

    public SearchProductsUseCase(ProductJpaRepository repo, ProductSearchMapper mapper) {
        this.repo = repo; this.mapper = mapper;
    }

    @Transactional(readOnly = true)
    public ProductListResponse search(ProductListRequest req) {
        var cursor = req.cursor() == null ? null : Cursor.decode(req.cursor());
        var sort = req.sort() == null ? SortKey.LATEST : req.sort();

        Specification<ProductJpaEntity> spec = Specification.where(ProductSpecs.activeOnly())
            .and(ProductSpecs.nameContains(req.q()))
            .and(ProductSpecs.brandIn(req.brand()))
            .and(ProductSpecs.categoryDescendantsOf(resolveCategoryPath(req.category())))
            .and(ProductSpecs.priceBetween(req.priceMin(), req.priceMax()))
            .and(ProductSpecs.hasAnyTag(req.tag()))
            .and(ProductSpecs.hasAnyOptionValue("Size", req.size()))
            .and(ProductSpecs.afterCursor(cursor, sort));

        // +1 트릭 — limit+1 만 조회해서 hasNext 결정
        var pageable = PageRequest.of(0, req.limit() + 1, ProductSpecs.toSort(sort));
        var raw = repo.findAll(spec, pageable).getContent();

        var hasNext = raw.size() > req.limit();
        var items = (hasNext ? raw.subList(0, req.limit()) : raw).stream()
            .map(mapper::toSummary)
            .toList();

        var nextCursor = hasNext
            ? buildCursor(raw.get(req.limit() - 1), sort).encode()
            : null;

        return new ProductListResponse(items, nextCursor, hasNext);
    }

    private String resolveCategoryPath(String categoryId) {
        if (categoryId == null) return null;
        return categoryRepo.findById(categoryId).orElseThrow().getPath();
    }
}
```

### 5.6 N+1 회피 — `@EntityGraph` + projection

```java
// ProductJpaRepository.java
public interface ProductJpaRepository
    extends JpaRepository<ProductJpaEntity, String>, JpaSpecificationExecutor<ProductJpaEntity> {

    @EntityGraph(attributePaths = {"brand", "images", "options"})
    @Override
    Page<ProductJpaEntity> findAll(Specification<ProductJpaEntity> spec, Pageable pageable);
}
```

> **함정 (N+1)**: 위 한 줄이 없으면 목록 20개 → 옵션 20번 + 이미지 20번 + 브랜드 20번 = **61 쿼리**. EntityGraph 로 **2~3 쿼리** 로 줄어듦.

> **함정 (cartesian product)**: 여러 `@OneToMany` 컬렉션을 한 쿼리에 `fetch join` 하면 row 폭발. **`@EntityGraph` 가 자동으로 분리** (Hibernate batch). 또는 `@BatchSize(size=20)`.

상세는 [[../pitfalls/n-plus-one]] 참고.

### 5.7 Projection — 목록은 필요 없는 컬럼 빼기

```java
// 상세 (Detail) 에는 description / images / options 모두.
// 목록 (Summary) 에는 description / 옵션 stock 등 불필요.

public interface ProductSummaryView {
    String getId();
    String getName();
    long getBasePriceKrw();
    String getMainImageUrl();
    Instant getCreatedAt();
    String getBrandId();
    String getBrandName();
    boolean getSoldOut();
}

@Query("""
    select
      p.id as id, p.name as name, p.basePriceKrw as basePriceKrw,
      p.mainImageUrl as mainImageUrl, p.createdAt as createdAt,
      b.id as brandId, b.name as brandName,
      case when sum(o.stock) = 0 then true else false end as soldOut
    from ProductJpaEntity p
      join p.brand b
      left join p.options o
    where p.status = 'ACTIVE'
    group by p.id, b.id, b.name
""")
List<ProductSummaryView> findSummaryView(...);
```

> **함정**: Projection 은 빠르지만 JPA 도메인 매핑 우회 → 두 가지 read path 가 생김. 일관성 신경 쓸 것.

### 5.8 Controller

```java
@RestController
@RequestMapping("/api/v1/products")
public class PublicProductSearchController {

    private final SearchProductsUseCase search;
    public PublicProductSearchController(SearchProductsUseCase search) { this.search = search; }

    @GetMapping
    public ApiResponse<ProductListResponse> list(@Valid ProductListRequest req) {
        return ApiResponse.ok(search.search(req));
    }
}
```

> **함정 (GET + multi-value param)**: Spring 은 `brand=a&brand=b` 를 `List<String>` 으로 자동 바인딩. 컬렉션 파라미터는 query string 으로 받기 (POST body X — RESTful 위반).

### 5.9 페이지 번호 페이지네이션 (관리자용)

```java
@GetMapping("/admin/products")
@PreAuthorize("hasRole('ADMIN')")
public ApiResponse<Page<ProductSummaryView>> adminList(
    @RequestParam(required = false) String q,
    @RequestParam(defaultValue = "0") @Min(0) @Max(1000) int page,
    @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
    @RequestParam(defaultValue = "createdAt,desc") String sort
) {
    var pageable = PageRequest.of(page, size, parseSort(sort));
    return ApiResponse.ok(repo.adminList(q, pageable));
}
```

> **함정**: offset 페이지네이션은 `count(*)` 비용 큼. **`@QueryHints(@QueryHint(name = "org.hibernate.timeout", value = "5"))`** 또는 `Slice` (count 없음) 활용.

---

## 6. 단계 3 — Elasticsearch 로 가야 할 때

### 6.1 신호

- PG 검색이 200ms 초과
- 자유 텍스트 (오타 / 동의어 / 한국어 분석) 필요
- 가중 랭킹 (boost) 필요
- 자동완성 / 인기 검색어
- fuzzy / 동의어 / synonym filter

### 6.2 구조

```
PG (진실의 원천) ──→ outbox / Debezium CDC ──→ Kafka ──→ ES 색인 워커
                                                              │
                                                              ▼
                                                       Elasticsearch
                                                              │
                                                              ▼ 검색 쿼리
                                                       ProductSearchService (ES 쿼리)
```

본 레시피 범위 밖. `elasticsearch-integration` 레시피 (예정) 에서.

---

## 7. 트랜잭션 / 예외 / 검증

### 7.1 read-only 트랜잭션

```java
@Transactional(readOnly = true)
public ProductListResponse search(...) { ... }
```

`readOnly = true` 는:
- Hibernate 의 dirty checking 생략 → 성능 ↑
- 일부 replica 라우팅에 hint (`@Transactional(readOnly = true)` + AbstractRoutingDataSource)

### 7.2 예외

| 예외 | 상태 | 코드 |
| --- | --- | --- |
| `BadRequestException("invalid cursor")` | 400 | `INVALID_CURSOR` |
| `MethodArgumentNotValidException` | 422 | `VALIDATION_FAILED` |
| `IllegalArgumentException` (정렬키) | 400 | `INVALID_SORT` |

---

## 8. 테스트

### 8.1 cursor 페이지네이션 — keyset 정확성

```java
@Test
void cursor_pagination_returns_consecutive_pages_without_gaps_or_duplicates() {
    // given — 100개 상품 (createdAt 균등)
    productFixture.createMany(100);

    // when — 5번 페이지 순회
    var seen = new HashSet<String>();
    String cursor = null;
    for (int i = 0; i < 5; i++) {
        var res = rest.getForEntity(
            "/api/v1/products?sort=latest&limit=20" + (cursor == null ? "" : "&cursor=" + cursor),
            Map.class);
        var data = (Map<?, ?>) res.getBody().get("data");
        var items = (List<Map<?, ?>>) data.get("items");
        items.forEach(it -> {
            var id = (String) it.get("productId");
            assertThat(seen).doesNotContain(id);    // 중복 X
            seen.add(id);
        });
        cursor = (String) data.get("nextCursor");
        if (cursor == null) break;
    }
    assertThat(seen).hasSize(100);                  // 누락 X
}
```

### 8.2 필터 결합

```java
@Test
void brand_and_price_filter_combined() {
    var acme = brandFixture.create("Acme");
    var beta = brandFixture.create("Beta");
    productFixture.create("후드1", acme, 50000);
    productFixture.create("후드2", acme, 90000);
    productFixture.create("후드3", beta, 50000);

    var res = rest.getForEntity(
        "/api/v1/products?brand=" + acme.id() + "&priceMax=60000", Map.class);
    var items = (List<?>) ((Map<?, ?>) res.getBody().get("data")).get("items");
    assertThat(items).hasSize(1);
}
```

### 8.3 N+1 검증

```java
@Test
void list_does_not_cause_N_plus_1() {
    productFixture.createMany(50);

    // Hibernate Statistics 활성화
    var stats = entityManager.getEntityManagerFactory().unwrap(SessionFactory.class).getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();

    rest.getForEntity("/api/v1/products?limit=50", Map.class);

    long queryCount = stats.getQueryExecutionCount();
    assertThat(queryCount).isLessThanOrEqualTo(5);   // 1 (main) + 1 (options) + 1 (images) + 1 (brand) + 1 (tags)
}
```

---

## 9. 운영 체크리스트

- [ ] `EXPLAIN ANALYZE` 로 자주 쓰는 검색 조합의 plan 확인 (Seq Scan X)
- [ ] 미사용 인덱스 정기 점검 (`pg_stat_user_indexes`)
- [ ] 검색 응답 p95 / p99 모니터링
- [ ] cursor 디코드 실패 시 400 (서버 에러 X)
- [ ] limit 최대값 강제 (100 권장) — DoS 방어
- [ ] readOnly 트랜잭션 + 가능하면 read replica 라우팅
- [ ] `product_stats` 갱신 job 라이브니스 알람
- [ ] EntityGraph / batch size 로 N+1 회피 확인
- [ ] 외부 노출 API 의 `count(*)` 비용 (필요 없으면 빼기)
- [ ] 검색 로그 (어떤 키워드가 0건 결과인가) — 개선 신호

---

## 10. 함정 모음

### 함정 1 — N+1
[[../pitfalls/n-plus-one]] 참고. 본 레시피 §5.6 의 `@EntityGraph` 가 필수.

### 함정 2 — offset 깊이
50페이지 + offset = 1000 부터 체감 느림. **cursor 기본** 정책.

### 함정 3 — 정렬 키에 unique 부족
`ORDER BY created_at DESC` 만으로는 같은 시간 항목이 매 페이지 섞임. **반드시 tie-breaker (id)**.

### 함정 4 — Specification null 반환
`null` 반환은 "조건 없음" 의미. 빈 list 가 들어와도 null 반환하면 무조건 통과. **빈 list 명시적 처리**.

### 함정 5 — 자유 텍스트 검색에 `LIKE %x%`
`%x%` 는 인덱스 사용 못함. **pg_trgm GIN** 또는 ES.

### 함정 6 — 필터 조합마다 인덱스
2^k 인덱스 = INSERT 죽음. **선택성 높은 컬럼만** + 나머지는 통계 기반.

### 함정 7 — Spring Data `Page` 의 `count(*)` 비용
큰 테이블 + 필터 = count 가 검색보다 느림. **`Slice` 또는 `+1 트릭`** (이 레시피).

### 함정 8 — 카테고리 자식 포함 안 함
`category = 'top'` 이면 `top/hoodie` 빠짐. **path prefix 매칭** (`path LIKE 'top/%'`).

### 함정 9 — `@OneToMany` 두 개 동시 fetch join
cartesian product. **`@EntityGraph` 사용** (Hibernate 가 자동 분리).

### 함정 10 — cursor 안에 만료 / 서명 없음
cursor 조작으로 다른 사용자 데이터 보기 가능 (필터에 user_id 가 묶여 있으면). **검증** 또는 HMAC 서명.

---

## 11. 관련

- [[product-crud]] — CRUD 전제
- [[../pitfalls/n-plus-one]] — N+1 깊이
- elasticsearch-integration (다음) — 단계 3
- cache-redis (다음) — hot 상품 캐시
- [[api-design|↑ api-design hub]]
