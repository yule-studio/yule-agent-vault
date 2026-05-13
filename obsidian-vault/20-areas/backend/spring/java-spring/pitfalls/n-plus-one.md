---
title: "N+1 Query Problem — 감지 / 회피 / 측정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:15:00+09:00
tags:
  - backend
  - java-spring
  - pitfalls
  - jpa
  - hibernate
  - performance
---

# N+1 Query Problem — 감지 / 회피 / 측정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 증상·감지·회피 5가지 + 비교 표 + 측정 |

**[[pitfalls|↑ pitfalls hub]]**

---

## 1. 증상

목록 N 개를 가져왔는데 SQL 이 **N+1 번** 나가는 현상.

```
SELECT * FROM products WHERE status = 'ACTIVE' LIMIT 20;       -- 1
SELECT * FROM brands WHERE id = ?                              -- N(20)
SELECT * FROM brands WHERE id = ?
...
SELECT * FROM product_options WHERE product_id = ?             -- N(20)
SELECT * FROM product_options WHERE product_id = ?
...
```

→ **20개 조회에 41+ 쿼리**. 네트워크 RTT × 쿼리 수.

---

## 2. 원인

JPA / Hibernate 의 `LAZY` 연관관계가 access 시점에 추가 쿼리 발사.

```java
@Entity
class ProductJpaEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    private BrandJpaEntity brand;          // LAZY

    @OneToMany(mappedBy = "product")
    private List<ProductOptionJpaEntity> options;   // LAZY by default
}

// 코드
var products = productRepo.findAll(spec, pageable);    // 1 쿼리
for (var p : products) {
    p.getBrand().getName();                            // → +1 per product
    p.getOptions().size();                             // → +1 per product
}
// 총 1 + 2N
```

**EAGER 로 바꿔도 해결 X** — 단일 fetch 는 OK, 컬렉션 fetch 는 cartesian product 폭발 (다음 §6).

---

## 3. 감지 — "내 코드에 N+1 있는가?"

### 3.1 Hibernate Statistics

```java
@Configuration
class JpaStatsConfig {
    @Bean
    public StatisticsConfig stats(EntityManagerFactory emf) {
        emf.unwrap(SessionFactory.class).getStatistics().setStatisticsEnabled(true);
        return new StatisticsConfig();
    }
}
```

```yaml
# application.yml — dev/staging
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session.events.log: true
```

런타임 측정:

```java
var stats = sessionFactory.getStatistics();
stats.clear();
service.list();
log.info("queries={}, entities={}",
    stats.getQueryExecutionCount(), stats.getEntityLoadCount());
```

### 3.2 SQL 추적 — `p6spy` / `datasource-proxy`

```kotlin
// build.gradle.kts
implementation("com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0")
```

```yaml
# application.yml — dev
decorator:
  datasource:
    p6spy:
      enable-logging: true
      multiline: true
```

로그 예시:
```
[QUERY] | took 5 ms | connection 0 | select ... from products ...
[QUERY] | took 2 ms | connection 0 | select ... from brands where id = 'B1'
[QUERY] | took 1 ms | connection 0 | select ... from brands where id = 'B2'
...
```

→ 같은 쿼리가 반복 = N+1 직감.

### 3.3 통합 테스트로 enforce — "쿼리 수 한계"

```java
@Test
void list_does_not_cause_N_plus_1() {
    productFixture.createMany(50);
    var stats = sessionFactory.getStatistics();
    stats.clear();

    service.list(new ProductListRequest(...));

    assertThat(stats.getQueryExecutionCount())
        .as("N+1 의심 — 쿼리 수가 너무 많음")
        .isLessThanOrEqualTo(5);     // main + collection 2~3 + brand
}
```

→ 회귀 막는 가장 확실한 방법. CI 에 포함.

### 3.4 운영 — APM / `pg_stat_statements`

```sql
-- 자주 실행되는 같은 쿼리 패턴 보기
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

calls 가 비정상적으로 큰 단순 select = N+1 흔적.

---

## 4. 회피 방법 — 5가지

### 회피 1 — `@EntityGraph` (1순위)

가장 깔끔. Spring Data JPA 메서드에 어노테이션.

```java
public interface ProductJpaRepository extends JpaRepository<ProductJpaEntity, String> {

    @EntityGraph(attributePaths = {"brand", "options", "images", "tags"})
    Optional<ProductJpaEntity> findWithCollectionsById(String id);

    @EntityGraph(attributePaths = {"brand", "options", "images"})
    @Override
    Page<ProductJpaEntity> findAll(Specification<ProductJpaEntity> spec, Pageable pageable);
}
```

- **단일 (`@ManyToOne`) 은 JOIN** 한 번에
- **컬렉션 (`@OneToMany`) 은 별도 select + IN(...)** 으로 자동 분리 (cartesian product 회피)

권장 — Spring Data JPA 와 가장 자연스러움.

### 회피 2 — JPQL `JOIN FETCH`

```java
@Query("""
    select p from ProductJpaEntity p
      join fetch p.brand
      left join fetch p.options
    where p.id = :id
""")
Optional<ProductJpaEntity> findWithBrandAndOptionsById(@Param("id") String id);
```

- 명시적, 제어 강함
- **컬렉션은 1개만** fetch join 가능 (둘 이상 = cartesian)
- pagination 과 어색함 (`distinct` + 메모리 페이징)

### 회피 3 — `@BatchSize` / `default_batch_fetch_size`

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

또는 엔티티 단위:
```java
@OneToMany(mappedBy = "product")
@BatchSize(size = 100)
private List<ProductOptionJpaEntity> options;
```

동작:
- LAZY 로 두고
- 첫 access 시 **`IN (:id1, :id2, ..., :id100)`** 으로 한 번에
- N+1 → N/100 + 1

→ 가장 손쉬운 글로벌 안전망. **모든 프로젝트에 기본 추천**.

### 회피 4 — 명시적 두 단계 쿼리

```java
public List<ProductSummary> list(...) {
    var page = repo.findActive(pageable);              // 1 쿼리
    var ids = page.stream().map(...).toList();
    var optionsByProduct = optionRepo.findByProductIdIn(ids).stream()
        .collect(groupingBy(o -> o.getProductId()));   // 1 쿼리
    return page.stream()
        .map(p -> toSummary(p, optionsByProduct.get(p.getId())))
        .toList();
}
```

장점: 완전한 제어. ORM 마법 없음.
단점: 코드 길어짐. 도메인 매핑 직접.

### 회피 5 — DTO Projection (목록은 매핑된 row 가 더 빠름)

```java
@Query("""
    select new com.example.shop.application.product.ProductSummaryDto(
        p.id, p.name, p.basePriceKrw, p.mainImageUrl,
        b.id, b.name, p.createdAt
    )
    from ProductJpaEntity p join p.brand b
    where p.status = 'ACTIVE'
""")
List<ProductSummaryDto> findActiveSummaries(Pageable pageable);
```

- 한 쿼리 + DTO 직접
- 옵션 / 이미지가 list 에 필요 없는 경우 최적
- 단점: detail 과 read path 가 두 종류

---

## 5. 어떤 방법을 언제

```
질문 1: 단일 (~1개) 조회인가?
├─ Yes → @EntityGraph + findById
└─ No (목록) →

질문 2: 목록에 컬렉션 (options / images) 다 필요한가?
├─ Yes → @EntityGraph (1순위) 또는 BatchSize
└─ No  → DTO Projection (5번)

질문 3: 컬렉션 2개 이상 필요한가?
├─ Yes → @EntityGraph (자동 분리) — JOIN FETCH 둘 X
└─ No  → JOIN FETCH 단일 컬렉션

베이스 정책: default_batch_fetch_size = 100 글로벌. 그래도 부족하면 개별 @EntityGraph.
```

---

## 6. cartesian product (N+1 의 사촌)

```java
@Query("""
    select p from ProductJpaEntity p
      left join fetch p.options
      left join fetch p.images        // 둘 다 컬렉션!
    where p.id = :id
""")
Optional<ProductJpaEntity> bad(@Param("id") String id);
```

옵션 5 × 이미지 10 = **50 row** 가 PG 에서 올라옴. 결과 객체는 옵션 5 + 이미지 10 인데 SQL row 는 50 = 메모리 낭비 + slow.

해결:
- `@EntityGraph` 가 자동으로 분리
- 또는 명시적 두 쿼리

---

## 7. 측정 예시 — 실제 차이

`products 1000 / brand 100 / option per product 5 / image per product 3` 데이터셋. 목록 20개 조회.

| 방법 | 쿼리 수 | latency (local PG) |
| --- | --- | --- |
| naive (아무것도 안 함) | **1 + 20×3 = 61** | ~120 ms |
| `@BatchSize=100` | 1 + 3 = **4** | ~35 ms |
| `@EntityGraph` | **3** | ~25 ms |
| DTO Projection | **1** | ~15 ms |

→ 단순 list 에서 **10배 차이**. 운영 트래픽에선 DB 부하 / 네트워크가 곱해짐.

---

## 8. 도메인 모델과의 분리

`Product` (도메인) 가 JPA 의 lazy proxy 영향 받지 않으려면 — [[../api-design/product-crud]] 처럼 **도메인 ↔ JPA 분리**:

- `Product` 는 평범한 POJO. lazy 아님.
- `JpaProductRepositoryAdapter.findById` 가 `EntityGraph` 로 fetch 한 뒤 `User.reconstitute(...)` 처럼 도메인 객체 생성.
- 도메인 코드 어디서도 N+1 위험 없음 (이미 다 fetch 된 상태).

```java
@Override
public Optional<Product> findById(ProductId id) {
    return spring.findWithCollectionsById(id.value())    // EntityGraph
        .map(this::toDomain);                            // 도메인 객체 생성
}
```

---

## 9. 함정 안의 함정

### 함정 1 — `@OneToMany(fetch = EAGER)` 로 해결
컬렉션 EAGER = cartesian + 항상 fetch. **LAZY + EntityGraph** 가 정석.

### 함정 2 — `Hibernate.initialize(entity.getOptions())` 호출
1+1 만 줄임. 목록 20개면 또 N+1. 진짜 해결 아님.

### 함정 3 — `@Transactional` 없는 곳에서 lazy access
`LazyInitializationException`. Open-In-View (OSIV) 가 가려주지만 — OSIV 는 안티패턴 (§참고 — [[transaction-pitfalls]]).

### 함정 4 — `default_batch_fetch_size` 만 켜고 안심
BatchSize 는 안전망. 핵심 read path 는 `@EntityGraph` / DTO 로 명시.

### 함정 5 — Spring Data JPA `Pageable` 에 `@EntityGraph` 컬렉션
페이지네이션 + 컬렉션 fetch = Hibernate 가 메모리에서 paginate (HHH000104 경고). **EntityGraph 의 컬렉션은 별도 select 로 분리** — Hibernate 가 알아서 처리하지만, JPQL `JOIN FETCH` + Pageable 은 위험.

### 함정 6 — DTO Projection 의 `new ...()` 의 fully-qualified name 오타
런타임 에러. IDE 가 잘 못 잡음. 테스트 필수.

### 함정 7 — JPA Cache / Second Level
1차 캐시 (영속성 컨텍스트) 와 헷갈리지 말 것. 같은 트랜잭션 내 같은 entity 재조회는 캐시되지만, N+1 의 문제는 **첫 fetch**.

### 함정 8 — `count(*)` 도 별도 쿼리
Spring Data `Page` 는 `findAll` + `count` 두 쿼리. count 가 비싼 경우 `Slice` 사용 (count 안 함).

---

## 10. 운영 체크리스트

- [ ] `hibernate.default_batch_fetch_size = 100` 글로벌 설정
- [ ] dev / staging 에 `p6spy` 또는 SQL 로그
- [ ] dev / staging 에 `generate_statistics = true`
- [ ] 통합 테스트에 "쿼리 수 한계" assertion 포함
- [ ] OSIV (`spring.jpa.open-in-view`) = **false** ([[transaction-pitfalls]])
- [ ] APM 에 SQL count / 평균 latency 모니터
- [ ] `pg_stat_statements` 정기 점검 (top 20 calls)
- [ ] 새 endpoint 마다 EXPLAIN 한 번
- [ ] 도메인 ↔ JPA 분리로 lazy 위험 격리

---

## 11. 관련

- [[pitfalls|↑ pitfalls hub]]
- [[transaction-pitfalls]] — OSIV / LazyInit
- [[../api-design/product-search]] — 본 회피책 적용 예
- [[../api-design/product-crud]] — 도메인 ↔ JPA 분리
