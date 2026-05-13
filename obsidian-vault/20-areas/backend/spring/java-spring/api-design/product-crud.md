---
title: "상품 CRUD — Java Spring Boot 레시피 (무신사 / 쿠팡 스타일)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - commerce
  - product
---

# 상품 CRUD — Java Spring Boot 레시피 (무신사 / 쿠팡 스타일)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Java 21 / Product+Option+Image / 권한 분리 / 전체 코드 |

**[[api-design|↑ api-design hub]]**

> **참고 모델**: 무신사 (브랜드+옵션+사이즈+이미지+상태), 쿠팡 (재고+가격+로켓배송), 네이버 쇼핑 (aggregator — 본 레시피 범위 밖).
> 본 레시피는 **단일 마켓플레이스 (셀러 + 관리자 + 구매자)** 기준.

---

## 1. 무엇을 만드는가

```
# 셀러 — 상품 등록 / 수정 / 삭제 / 본인 상품 조회
POST   /api/v1/seller/products
GET    /api/v1/seller/products/{id}
PATCH  /api/v1/seller/products/{id}
DELETE /api/v1/seller/products/{id}            # soft delete

# 공개 — 구매자가 상품 보기
GET    /api/v1/products/{id}                   # 상세
GET    /api/v1/products                        # 목록 (다음 레시피 search 에서 깊이)
```

### 1.1 요청 / 응답 — 상품 등록

```http
POST /api/v1/seller/products
Authorization: Bearer ...
{
  "name": "오버사이즈 후드 — 차콜",
  "brandId": "01HZBRA...",
  "categoryId": "01HZCAT...",
  "basePriceKrw": 49000,
  "description": "...",
  "images": [
    { "url": "https://cdn.../1.jpg", "sort": 0, "kind": "MAIN" },
    { "url": "https://cdn.../2.jpg", "sort": 1, "kind": "DETAIL" }
  ],
  "options": [
    { "name": "Size", "value": "S", "extraPriceKrw": 0,    "stock": 30 },
    { "name": "Size", "value": "M", "extraPriceKrw": 0,    "stock": 50 },
    { "name": "Size", "value": "L", "extraPriceKrw": 0,    "stock": 30 },
    { "name": "Size", "value": "XL", "extraPriceKrw": 2000, "stock": 20 }
  ],
  "tags": ["오버사이즈", "스트릿"]
}

201 Created
{ "data": { "productId": "01HZPRD...", "status": "DRAFT" } }
```

### 1.2 비기능

- **권한**: 셀러는 본인 상품만 / 관리자는 전체
- **soft delete**: 삭제는 `status = DISCONTINUED` 또는 `deleted_at`. 주문 이력과 FK 관계 보존
- **이미지 / 옵션은 자식 컬렉션** — 별도 테이블, cascade
- **재고는 옵션 단위** — 옵션 없는 상품도 있어야 함 (단일 옵션 기본)
- **status 전이는 명확한 규칙** — DRAFT → ACTIVE → SOLD_OUT / DISCONTINUED
- **검색 색인 동기화** — 변경 시 Elasticsearch 색인 이벤트 (다음 레시피)

---

## 2. 도메인 모델

### 2.1 클래스 다이어그램

```
┌───────────────────────────┐
│ Product (Aggregate Root)  │
│ - id, sellerId, brandId   │
│ - categoryId              │
│ - name, description       │
│ - basePrice (Money)       │
│ - status (ProductStatus)  │
│ - createdAt, updatedAt    │
│ + addOption(...)          │
│ + removeOption(...)       │
│ + addImage(...)           │
│ + activate()              │
│ + discontinue()           │
│ + changePrice(Money)      │
└──────┬────────────────────┘
       │ 1..N
       ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│ ProductOption            │    │ ProductImage             │
│ - id, name, value        │    │ - id, url, sort, kind    │
│ - extraPrice (Money)     │    │ - kind: MAIN/DETAIL/...  │
│ - stock                  │    └──────────────────────────┘
│ - status                 │
└──────────────────────────┘

(별도 Aggregate)
Brand: id, name, slug, logoUrl
Category: id, parentId, name, slug  (트리)
Seller: User 의 role
```

### 2.2 Value Objects

```java
// src/main/java/com/example/shop/domain/common/Money.java
public record Money(long amountMinorUnit, Currency currency) {
    public Money {
        if (amountMinorUnit < 0) throw new IllegalArgumentException("negative money");
        if (currency == null) throw new IllegalArgumentException("currency required");
    }
    public static Money krw(long won) { return new Money(won, Currency.getInstance("KRW")); }
    public Money add(Money other) {
        ensureSame(other);
        return new Money(amountMinorUnit + other.amountMinorUnit, currency);
    }
    public Money multiply(int n) { return new Money(amountMinorUnit * n, currency); }
    private void ensureSame(Money o) {
        if (!currency.equals(o.currency))
            throw new IllegalArgumentException("currency mismatch");
    }
}

// src/main/java/com/example/shop/domain/product/ProductId.java
public record ProductId(String value) {
    public ProductId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("must be ULID");
    }
}
// BrandId, CategoryId, ProductOptionId, ProductImageId 도 동일 패턴

// src/main/java/com/example/shop/domain/product/ProductStatus.java
public enum ProductStatus {
    DRAFT,          // 셀러 작성 중, 비공개
    ACTIVE,         // 판매중
    SOLD_OUT,       // 재고 0 (자동 전이)
    DISCONTINUED,   // 단종 (셀러가 명시적)
    DELETED         // soft delete (관리자)
}
```

### 2.3 Product Aggregate

```java
// src/main/java/com/example/shop/domain/product/Product.java
public final class Product {

    private final ProductId id;
    private final UserId sellerId;
    private final BrandId brandId;
    private CategoryId categoryId;
    private String name;
    private String description;
    private Money basePrice;
    private ProductStatus status;
    private final Instant createdAt;
    private Instant updatedAt;

    private final List<ProductOption> options = new ArrayList<>();
    private final List<ProductImage> images = new ArrayList<>();
    private final Set<String> tags = new LinkedHashSet<>();
    private final List<DomainEvent> events = new ArrayList<>();

    private Product(ProductId id, UserId sellerId, BrandId brandId, CategoryId categoryId,
                    String name, String description, Money basePrice,
                    ProductStatus status, Instant createdAt, Instant updatedAt) {
        this.id = id; this.sellerId = sellerId; this.brandId = brandId;
        this.categoryId = categoryId; this.name = name; this.description = description;
        this.basePrice = basePrice; this.status = status;
        this.createdAt = createdAt; this.updatedAt = updatedAt;
    }

    public static Product create(ProductId id, UserId sellerId, BrandId brandId,
                                 CategoryId categoryId, String name, String description,
                                 Money basePrice, Instant now) {
        validateName(name);
        if (basePrice == null) throw new IllegalArgumentException("basePrice required");
        var p = new Product(id, sellerId, brandId, categoryId, name, description,
                            basePrice, ProductStatus.DRAFT, now, now);
        p.events.add(new ProductCreated(id, sellerId, now));
        return p;
    }

    // ---- Behavior ----

    public void rename(String newName, Instant now) {
        ensureMutable();
        validateName(newName);
        this.name = newName;
        this.updatedAt = now;
        events.add(new ProductUpdated(id, now));
    }

    public void changePrice(Money newPrice, Instant now) {
        ensureMutable();
        if (newPrice == null) throw new IllegalArgumentException("price required");
        this.basePrice = newPrice;
        this.updatedAt = now;
        events.add(new ProductUpdated(id, now));
    }

    public void changeCategory(CategoryId newCategoryId, Instant now) {
        ensureMutable();
        this.categoryId = newCategoryId;
        this.updatedAt = now;
        events.add(new ProductUpdated(id, now));
    }

    public ProductOption addOption(ProductOptionId optionId, String name, String value,
                                   Money extraPrice, int stock) {
        ensureMutable();
        if (options.size() >= 100) throw new IllegalStateException("too many options");
        if (options.stream().anyMatch(o -> o.matches(name, value)))
            throw new DuplicateOptionException(name, value);
        var opt = ProductOption.create(optionId, name, value, extraPrice, stock);
        options.add(opt);
        return opt;
    }

    public void removeOption(ProductOptionId optionId) {
        ensureMutable();
        var removed = options.removeIf(o -> o.id().equals(optionId));
        if (!removed) throw new NotFoundException();
    }

    public ProductImage addImage(ProductImageId imageId, String url, int sort,
                                 ProductImageKind kind) {
        ensureMutable();
        if (images.size() >= 30) throw new IllegalStateException("too many images");
        if (kind == ProductImageKind.MAIN && hasMain())
            throw new IllegalStateException("MAIN image already exists");
        var img = new ProductImage(imageId, url, sort, kind);
        images.add(img);
        return img;
    }

    public void activate(Instant now) {
        if (status != ProductStatus.DRAFT && status != ProductStatus.SOLD_OUT)
            throw new IllegalStateException("cannot activate from " + status);
        if (images.stream().noneMatch(i -> i.kind() == ProductImageKind.MAIN))
            throw new IllegalStateException("MAIN image required");
        if (totalStock() == 0) throw new IllegalStateException("no stock");
        this.status = ProductStatus.ACTIVE;
        this.updatedAt = now;
        events.add(new ProductActivated(id, now));
    }

    public void discontinue(Instant now) {
        if (status == ProductStatus.DELETED) throw new IllegalStateException("deleted");
        this.status = ProductStatus.DISCONTINUED;
        this.updatedAt = now;
        events.add(new ProductDiscontinued(id, now));
    }

    public void markDeleted(Instant now) {
        this.status = ProductStatus.DELETED;
        this.updatedAt = now;
        events.add(new ProductDeleted(id, now));
    }

    /** 옵션 재고 변동 후 재고 0 이면 SOLD_OUT 자동 전이. */
    public void recalculateStockStatus(Instant now) {
        if (status == ProductStatus.ACTIVE && totalStock() == 0) {
            this.status = ProductStatus.SOLD_OUT;
            this.updatedAt = now;
            events.add(new ProductSoldOut(id, now));
        }
    }

    // ---- queries ----

    public int totalStock() { return options.stream().mapToInt(ProductOption::stock).sum(); }
    public boolean hasMain() { return images.stream().anyMatch(i -> i.kind() == ProductImageKind.MAIN); }
    public boolean isOwnedBy(UserId user) { return sellerId.equals(user); }

    // getters
    public ProductId id() { return id; }
    public UserId sellerId() { return sellerId; }
    public BrandId brandId() { return brandId; }
    public CategoryId categoryId() { return categoryId; }
    public String name() { return name; }
    public String description() { return description; }
    public Money basePrice() { return basePrice; }
    public ProductStatus status() { return status; }
    public Instant createdAt() { return createdAt; }
    public Instant updatedAt() { return updatedAt; }
    public List<ProductOption> options() { return List.copyOf(options); }
    public List<ProductImage> images() { return List.copyOf(images); }
    public Set<String> tags() { return Set.copyOf(tags); }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events); events.clear(); return copy;
    }

    private void ensureMutable() {
        if (status == ProductStatus.DELETED || status == ProductStatus.DISCONTINUED)
            throw new IllegalStateException("immutable in status: " + status);
    }
    private static void validateName(String n) {
        if (n == null || n.isBlank() || n.length() > 200)
            throw new IllegalArgumentException("invalid product name");
    }

    public static Product reconstitute(ProductId id, UserId sellerId, BrandId brandId,
                                       CategoryId categoryId, String name, String description,
                                       Money basePrice, ProductStatus status,
                                       Instant createdAt, Instant updatedAt,
                                       List<ProductOption> opts, List<ProductImage> imgs,
                                       Set<String> tags) {
        var p = new Product(id, sellerId, brandId, categoryId, name, description,
                            basePrice, status, createdAt, updatedAt);
        p.options.addAll(opts);
        p.images.addAll(imgs);
        p.tags.addAll(tags);
        return p;
    }
}
```

### 2.4 ProductOption

```java
// src/main/java/com/example/shop/domain/product/ProductOption.java
public final class ProductOption {

    private final ProductOptionId id;
    private final String name;       // e.g. "Size"
    private final String value;      // e.g. "M"
    private Money extraPrice;
    private int stock;

    private ProductOption(ProductOptionId id, String name, String value,
                          Money extraPrice, int stock) {
        if (stock < 0) throw new IllegalArgumentException("negative stock");
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name required");
        this.id = id; this.name = name; this.value = value;
        this.extraPrice = extraPrice; this.stock = stock;
    }

    public static ProductOption create(ProductOptionId id, String name, String value,
                                       Money extraPrice, int stock) {
        return new ProductOption(id, name, value, extraPrice, stock);
    }

    public void decreaseStock(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty must be positive");
        if (stock < qty) throw new InsufficientStockException(id, qty, stock);
        stock -= qty;
    }
    public void increaseStock(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty must be positive");
        stock += qty;
    }

    public boolean matches(String n, String v) {
        return Objects.equals(name, n) && Objects.equals(value, v);
    }

    public ProductOptionId id() { return id; }
    public String name() { return name; }
    public String value() { return value; }
    public Money extraPrice() { return extraPrice; }
    public int stock() { return stock; }
}
```

> **재고 감소** 은 일단 메서드만. 동시성 처리 (낙관/비관/분산락) 는 [[order-stock]] 레시피에서 깊이.

### 2.5 ProductImage / Domain Events

```java
public enum ProductImageKind { MAIN, DETAIL, SWATCH }
public record ProductImage(ProductImageId id, String url, int sort, ProductImageKind kind) {
    public ProductImage {
        if (url == null || !url.startsWith("https://"))
            throw new IllegalArgumentException("image url must be https");
        if (sort < 0) throw new IllegalArgumentException("sort must be >= 0");
    }
}

public sealed interface DomainEvent
    permits UserRegistered, UserEmailVerified, UserPasswordChanged,
            ProductCreated, ProductUpdated, ProductActivated,
            ProductSoldOut, ProductDiscontinued, ProductDeleted {
    Instant occurredAt();
}

public record ProductCreated(ProductId productId, UserId sellerId, Instant occurredAt) implements DomainEvent {}
public record ProductUpdated(ProductId productId, Instant occurredAt) implements DomainEvent {}
public record ProductActivated(ProductId productId, Instant occurredAt) implements DomainEvent {}
public record ProductSoldOut(ProductId productId, Instant occurredAt) implements DomainEvent {}
public record ProductDiscontinued(ProductId productId, Instant occurredAt) implements DomainEvent {}
public record ProductDeleted(ProductId productId, Instant occurredAt) implements DomainEvent {}
```

### 2.6 Repository

```java
// src/main/java/com/example/shop/domain/product/ProductRepository.java
public interface ProductRepository {
    Product save(Product product);
    Optional<Product> findById(ProductId id);
    Optional<Product> findByIdForUpdate(ProductId id);     // 비관 락 (재고 감소에 사용)
    Page<Product> findActive(Pageable pageable);            // 단순 목록
}
```

---

## 3. DB 스키마

```sql
-- V10__create_brands.sql
CREATE TABLE brands (
  id          CHAR(26) PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(120) NOT NULL,
  logo_url    VARCHAR(500),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_brands_slug ON brands (slug);

-- V11__create_categories.sql
CREATE TABLE categories (
  id          CHAR(26) PRIMARY KEY,
  parent_id   CHAR(26) REFERENCES categories(id),
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(120) NOT NULL,
  depth       SMALLINT NOT NULL,
  path        VARCHAR(500) NOT NULL,                       -- '/men/top/hoodie'
  sort        INTEGER NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_categories_slug ON categories (slug);
CREATE INDEX ix_categories_parent ON categories (parent_id);
CREATE INDEX ix_categories_path ON categories (path text_pattern_ops);

-- V12__create_products.sql
CREATE TABLE products (
  id              CHAR(26) PRIMARY KEY,
  seller_id       CHAR(26) NOT NULL REFERENCES users(id),
  brand_id        CHAR(26) NOT NULL REFERENCES brands(id),
  category_id     CHAR(26) NOT NULL REFERENCES categories(id),
  name            VARCHAR(200) NOT NULL,
  description     TEXT,
  base_price_krw  BIGINT NOT NULL CHECK (base_price_krw >= 0),
  status          VARCHAR(20) NOT NULL,
  version         BIGINT NOT NULL DEFAULT 0,                -- @Version (낙관 락)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_products_seller ON products (seller_id);
CREATE INDEX ix_products_brand_status ON products (brand_id, status);
CREATE INDEX ix_products_category_status ON products (category_id, status);
CREATE INDEX ix_products_updated_at ON products (updated_at DESC);

-- V13__create_product_options.sql
CREATE TABLE product_options (
  id              CHAR(26) PRIMARY KEY,
  product_id      CHAR(26) NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  name            VARCHAR(50) NOT NULL,
  value           VARCHAR(100) NOT NULL,
  extra_price_krw BIGINT NOT NULL DEFAULT 0 CHECK (extra_price_krw >= 0),
  stock           INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
  version         BIGINT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_product_options_unique ON product_options (product_id, name, value);
CREATE INDEX ix_product_options_product ON product_options (product_id);

-- V14__create_product_images.sql
CREATE TABLE product_images (
  id          CHAR(26) PRIMARY KEY,
  product_id  CHAR(26) NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  url         VARCHAR(500) NOT NULL,
  sort        INTEGER NOT NULL DEFAULT 0,
  kind        VARCHAR(20) NOT NULL,                         -- MAIN / DETAIL / SWATCH
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_product_images_product ON product_images (product_id, sort);

-- V15__create_product_tags.sql
CREATE TABLE product_tags (
  product_id  CHAR(26) NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  tag         VARCHAR(50) NOT NULL,
  PRIMARY KEY (product_id, tag)
);
CREATE INDEX ix_product_tags_tag ON product_tags (tag);
```

> **함정**: 카테고리 트리를 self-referencing FK 만으로 운영하면 깊은 트리 쿼리 어려움. `path` 컬럼 (`/men/top/hoodie`) 또는 ltree extension 권장.

---

## 4. 권한 / 인가

### 4.1 Role 매트릭스

| Endpoint | BUYER | SELLER (본인) | SELLER (타인) | ADMIN |
| --- | --- | --- | --- | --- |
| `POST /seller/products` | ❌ | ✅ | n/a | ✅ |
| `PATCH /seller/products/{id}` | ❌ | ✅ | ❌ | ✅ |
| `DELETE /seller/products/{id}` | ❌ | ✅ (soft) | ❌ | ✅ |
| `GET /seller/products/{id}` | ❌ | ✅ | ❌ | ✅ |
| `GET /products/{id}` (ACTIVE 만) | ✅ | ✅ | ✅ | ✅ |
| `GET /products/{id}` (DRAFT) | ❌ | ✅ (본인) | ❌ | ✅ |

### 4.2 Spring Security `@PreAuthorize` + 서비스 체크

```java
// 컨트롤러 단 — role 만 빠르게
@PreAuthorize("hasRole('SELLER') or hasRole('ADMIN')")
@PostMapping("/seller/products")
public ApiResponse<...> create(...) { ... }

// 서비스 단 — ownership 체크 (도메인 로직)
if (!product.isOwnedBy(currentUserId) && !isAdmin) throw new ForbiddenException();
```

> **함정**: ownership 체크를 Controller 의 `@PreAuthorize` SpEL 에 넣으면 도메인 로직이 새는 위치 분산. **Service 가 진실의 원천**.

---

## 5. 구현 — Spring Boot

### 5.1 패키지 구조

```
com.example.shop
├── domain/product/
│   ├── Product.java
│   ├── ProductOption.java
│   ├── ProductImage.java
│   ├── ProductStatus.java
│   ├── Money.java
│   ├── ProductRepository.java
│   └── events/*.java
├── application/product/
│   ├── CreateProductUseCase.java
│   ├── UpdateProductUseCase.java
│   ├── DeleteProductUseCase.java
│   ├── GetProductUseCase.java
│   └── commands/*.java
├── infrastructure/persistence/jpa/product/
│   ├── ProductJpaEntity.java
│   ├── ProductOptionJpaEntity.java
│   ├── ProductImageJpaEntity.java
│   ├── ProductTagJpaEntity.java
│   ├── ProductJpaRepository.java
│   └── JpaProductRepositoryAdapter.java
└── presentation/api/v1/
    ├── seller/SellerProductController.java
    └── product/PublicProductController.java
```

### 5.2 Command / DTO

```java
// src/main/java/com/example/shop/application/product/commands/CreateProductCommand.java
public record CreateProductCommand(
    UserId sellerId,
    BrandId brandId,
    CategoryId categoryId,
    String name,
    String description,
    Money basePrice,
    List<CreateOption> options,
    List<CreateImage> images,
    List<String> tags
) {
    public record CreateOption(String name, String value, Money extraPrice, int stock) {}
    public record CreateImage(String url, int sort, ProductImageKind kind) {}
}

// presentation/api/v1/seller/dto
public record CreateProductRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank String brandId,
    @NotBlank String categoryId,
    @PositiveOrZero long basePriceKrw,
    String description,
    @Valid @NotNull List<ImageDto> images,
    @Valid List<OptionDto> options,
    List<@Size(max = 50) String> tags
) {
    public record OptionDto(
        @NotBlank @Size(max = 50) String name,
        @NotBlank @Size(max = 100) String value,
        @PositiveOrZero long extraPriceKrw,
        @PositiveOrZero int stock
    ) {}
    public record ImageDto(
        @NotBlank @Pattern(regexp = "^https://.*") String url,
        @PositiveOrZero int sort,
        @NotNull ProductImageKind kind
    ) {}
}
```

### 5.3 CreateProductUseCase

```java
// src/main/java/com/example/shop/application/product/CreateProductUseCase.java
@Service
public class CreateProductUseCase {

    private final ProductRepository products;
    private final BrandRepository brands;
    private final CategoryRepository categories;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    public CreateProductUseCase(ProductRepository products, BrandRepository brands,
                                CategoryRepository categories, IdGenerator ids,
                                Clock clock, ApplicationEventPublisher events) {
        this.products = products; this.brands = brands;
        this.categories = categories; this.ids = ids;
        this.clock = clock; this.events = events;
    }

    @Transactional
    public ProductId handle(CreateProductCommand cmd) {
        // 참조 무결성 검증 (FK 도 잡지만 친절한 메시지)
        brands.findById(cmd.brandId()).orElseThrow(() -> new NotFoundException("brand"));
        categories.findById(cmd.categoryId()).orElseThrow(() -> new NotFoundException("category"));

        var now = Instant.now(clock);
        var product = Product.create(
            new ProductId(ids.next()),
            cmd.sellerId(), cmd.brandId(), cmd.categoryId(),
            cmd.name(), cmd.description(), cmd.basePrice(),
            now
        );

        cmd.options().forEach(o ->
            product.addOption(
                new ProductOptionId(ids.next()),
                o.name(), o.value(), o.extraPrice(), o.stock()
            )
        );
        cmd.images().forEach(i ->
            product.addImage(new ProductImageId(ids.next()), i.url(), i.sort(), i.kind())
        );

        var saved = products.save(product);
        saved.pullDomainEvents().forEach(events::publishEvent);
        return saved.id();
    }
}
```

### 5.4 UpdateProductUseCase (옵션 / 이미지 교체 포함)

```java
// src/main/java/com/example/shop/application/product/UpdateProductUseCase.java
@Service
public class UpdateProductUseCase {

    private final ProductRepository products;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    public UpdateProductUseCase(ProductRepository products, IdGenerator ids,
                                Clock clock, ApplicationEventPublisher events) {
        this.products = products; this.ids = ids;
        this.clock = clock; this.events = events;
    }

    @Transactional
    public void handle(UpdateProductCommand cmd) {
        var product = products.findById(cmd.productId())
            .orElseThrow(() -> new NotFoundException("product"));

        if (!product.isOwnedBy(cmd.sellerId()) && !cmd.isAdmin())
            throw new ForbiddenException();

        var now = Instant.now(clock);
        cmd.newName().ifPresent(n -> product.rename(n, now));
        cmd.newPrice().ifPresent(p -> product.changePrice(p, now));
        cmd.newCategoryId().ifPresent(c -> product.changeCategory(c, now));

        // 옵션 교체 — 단순 구현 (전체 교체). 부분 수정은 별도 endpoint 권장.
        if (cmd.newOptions().isPresent()) {
            // 기존 옵션 모두 제거 후 신규 추가
            product.options().forEach(o -> product.removeOption(o.id()));
            cmd.newOptions().get().forEach(o ->
                product.addOption(new ProductOptionId(ids.next()),
                                  o.name(), o.value(), o.extraPrice(), o.stock())
            );
        }

        products.save(product);
        product.pullDomainEvents().forEach(events::publishEvent);
    }

    @Transactional
    public void activate(ProductId id, UserId sellerId, boolean isAdmin) {
        var product = products.findById(id).orElseThrow(NotFoundException::new);
        if (!product.isOwnedBy(sellerId) && !isAdmin) throw new ForbiddenException();
        product.activate(Instant.now(clock));
        products.save(product);
        product.pullDomainEvents().forEach(events::publishEvent);
    }

    @Transactional
    public void discontinue(ProductId id, UserId sellerId, boolean isAdmin) {
        var product = products.findById(id).orElseThrow(NotFoundException::new);
        if (!product.isOwnedBy(sellerId) && !isAdmin) throw new ForbiddenException();
        product.discontinue(Instant.now(clock));
        products.save(product);
        product.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

### 5.5 DeleteProductUseCase (soft delete)

```java
@Service
public class DeleteProductUseCase {
    private final ProductRepository products;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    public DeleteProductUseCase(ProductRepository products, Clock clock,
                                ApplicationEventPublisher events) {
        this.products = products; this.clock = clock; this.events = events;
    }

    @Transactional
    public void handle(ProductId id, UserId sellerId, boolean isAdmin) {
        var product = products.findById(id).orElseThrow(NotFoundException::new);
        if (!product.isOwnedBy(sellerId) && !isAdmin) throw new ForbiddenException();
        // 주문 / 장바구니에 있는 상품도 ID 참조 유지 위해 hard delete X
        product.markDeleted(Instant.now(clock));
        products.save(product);
        product.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

### 5.6 Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/seller/SellerProductController.java
@RestController
@RequestMapping("/api/v1/seller/products")
@PreAuthorize("hasAnyRole('SELLER', 'ADMIN')")
public class SellerProductController {

    private final CreateProductUseCase create;
    private final UpdateProductUseCase update;
    private final DeleteProductUseCase delete;
    private final GetProductUseCase getProduct;

    public SellerProductController(CreateProductUseCase create, UpdateProductUseCase update,
                                   DeleteProductUseCase delete, GetProductUseCase getProduct) {
        this.create = create; this.update = update;
        this.delete = delete; this.getProduct = getProduct;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<Map<String, String>> create(
        @Valid @RequestBody CreateProductRequest req,
        Authentication auth
    ) {
        var cmd = toCommand(req, new UserId(auth.getName()));
        var id = create.handle(cmd);
        return ApiResponse.ok(Map.of("productId", id.value()));
    }

    @PatchMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void update(
        @PathVariable String id,
        @Valid @RequestBody UpdateProductRequest req,
        Authentication auth
    ) {
        update.handle(toUpdateCommand(req, new ProductId(id),
                                      new UserId(auth.getName()), isAdmin(auth)));
    }

    @PostMapping("/{id}/activate")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void activate(@PathVariable String id, Authentication auth) {
        update.activate(new ProductId(id), new UserId(auth.getName()), isAdmin(auth));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable String id, Authentication auth) {
        delete.handle(new ProductId(id), new UserId(auth.getName()), isAdmin(auth));
    }

    @GetMapping("/{id}")
    public ApiResponse<ProductDetailResponse> get(@PathVariable String id, Authentication auth) {
        return ApiResponse.ok(getProduct.forSeller(
            new ProductId(id), new UserId(auth.getName()), isAdmin(auth)));
    }

    private static boolean isAdmin(Authentication auth) {
        return auth.getAuthorities().stream().anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    }

    private static CreateProductCommand toCommand(CreateProductRequest r, UserId sellerId) {
        var opts = r.options() == null ? List.<CreateProductCommand.CreateOption>of()
            : r.options().stream().map(o -> new CreateProductCommand.CreateOption(
                o.name(), o.value(), Money.krw(o.extraPriceKrw()), o.stock())).toList();
        var imgs = r.images().stream().map(i -> new CreateProductCommand.CreateImage(
                i.url(), i.sort(), i.kind())).toList();
        return new CreateProductCommand(
            sellerId,
            new BrandId(r.brandId()),
            new CategoryId(r.categoryId()),
            r.name(), r.description(), Money.krw(r.basePriceKrw()),
            opts, imgs, r.tags() == null ? List.of() : r.tags()
        );
    }

    private static UpdateProductCommand toUpdateCommand(UpdateProductRequest r, ProductId id,
                                                        UserId sellerId, boolean admin) {
        // 필드별 Optional 변환 — 생략
        throw new UnsupportedOperationException("mapping");
    }
}

// 공개 API
@RestController
@RequestMapping("/api/v1/products")
public class PublicProductController {

    private final GetProductUseCase getProduct;
    public PublicProductController(GetProductUseCase getProduct) { this.getProduct = getProduct; }

    @GetMapping("/{id}")
    public ApiResponse<ProductDetailResponse> get(@PathVariable String id) {
        return ApiResponse.ok(getProduct.forPublic(new ProductId(id)));
    }
}
```

### 5.7 응답 DTO

```java
public record ProductDetailResponse(
    String productId,
    String name,
    String description,
    long basePriceKrw,
    String status,
    BrandSummary brand,
    CategorySummary category,
    List<ImageDto> images,
    List<OptionDto> options,
    List<String> tags,
    Instant createdAt,
    Instant updatedAt
) {
    public record BrandSummary(String id, String name, String slug) {}
    public record CategorySummary(String id, String name, String path) {}
    public record OptionDto(String id, String name, String value, long extraPriceKrw, int stock) {}
    public record ImageDto(String url, int sort, String kind) {}
}
```

### 5.8 JPA Entity (스케치 — 길어서 핵심만)

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/product/ProductJpaEntity.java
@Entity
@Table(name = "products")
public class ProductJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "seller_id", nullable = false, length = 26) private String sellerId;
    @Column(name = "brand_id", nullable = false, length = 26) private String brandId;
    @Column(name = "category_id", nullable = false, length = 26) private String categoryId;
    @Column(nullable = false, length = 200) private String name;
    @Column(columnDefinition = "text") private String description;
    @Column(name = "base_price_krw", nullable = false) private long basePriceKrw;
    @Column(nullable = false, length = 20) private String status;

    @Version @Column(nullable = false) private long version;
    @Column(name = "created_at", nullable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private final List<ProductOptionJpaEntity> options = new ArrayList<>();

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("sort ASC")
    private final List<ProductImageJpaEntity> images = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "product_tags", joinColumns = @JoinColumn(name = "product_id"))
    @Column(name = "tag")
    private final Set<String> tags = new LinkedHashSet<>();

    protected ProductJpaEntity() {}
    // 생성자 + 매핑 메서드 생략
}
```

> **함정 (N+1)**: `findById` 후 `images` / `options` 접근 시 lazy 로딩 = 쿼리 폭발.
> 해결: `@EntityGraph(attributePaths = {"options", "images", "tags"})` 또는 명시적 `fetch join`.

```java
public interface ProductJpaRepository extends JpaRepository<ProductJpaEntity, String> {
    @EntityGraph(attributePaths = {"options", "images", "tags"})
    Optional<ProductJpaEntity> findWithCollectionsById(String id);

    // 비관 락 — 재고 차감 시
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from ProductJpaEntity p where p.id = :id")
    Optional<ProductJpaEntity> findByIdForUpdate(@Param("id") String id);
}
```

---

## 6. 트랜잭션 / 예외 / 검증

### 6.1 예외 매핑

| 예외 | 상태 | 코드 |
| --- | --- | --- |
| `NotFoundException` | 404 | `PRODUCT_NOT_FOUND` / `BRAND_NOT_FOUND` / ... |
| `ForbiddenException` | 403 | `FORBIDDEN` |
| `DuplicateOptionException` | 409 | `DUPLICATE_OPTION` |
| `IllegalStateException` (도메인) | 422 | `INVALID_STATE_TRANSITION` |
| `OptimisticLockException` | 409 | `CONCURRENT_UPDATE` (다시 시도하세요) |
| `MethodArgumentNotValidException` | 422 | `VALIDATION_FAILED` |

### 6.2 낙관 락

- `@Version` 컬럼이 자동 감지
- `OptimisticLockException` 가 던져지면 클라이언트에 409 + 재시도 안내
- 대안: 비관 락은 **재고 감소 같은 짧고 빈번한 변경** 에만

---

## 7. 테스트

### 7.1 도메인 단위

```java
class ProductTest {

    @Test
    void create_initial_status_is_DRAFT() {
        var p = createValidProduct();
        assertThat(p.status()).isEqualTo(ProductStatus.DRAFT);
        assertThat(p.pullDomainEvents()).hasAtLeastOneElementOfType(ProductCreated.class);
    }

    @Test
    void cannot_activate_without_MAIN_image() {
        var p = createValidProduct();
        p.addOption(new ProductOptionId(ulid()), "Size", "M", Money.krw(0), 10);
        assertThatThrownBy(() -> p.activate(Instant.now()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("MAIN image");
    }

    @Test
    void duplicate_option_throws() {
        var p = createValidProduct();
        p.addOption(new ProductOptionId(ulid()), "Size", "M", Money.krw(0), 10);
        assertThatThrownBy(() -> p.addOption(new ProductOptionId(ulid()),
                                             "Size", "M", Money.krw(0), 10))
            .isInstanceOf(DuplicateOptionException.class);
    }

    @Test
    void stock_zero_after_activate_transitions_to_SOLD_OUT() {
        var p = createActiveProductWithStock(5);
        var optId = p.options().get(0).id();
        p.options().get(0).decreaseStock(5);
        p.recalculateStockStatus(Instant.now());
        assertThat(p.status()).isEqualTo(ProductStatus.SOLD_OUT);
    }
}
```

### 7.2 통합

```java
@Test
void seller_can_create_get_update_delete_own_product() {
    var seller = userFixture.createActiveSeller("seller1@x.com");
    var auth = "Bearer " + tokens.access(seller);
    var brand = brandFixture.create("Acme");
    var category = categoryFixture.create("Top/Hoodie");

    // CREATE
    var createRes = rest.exchange("/api/v1/seller/products",
        HttpMethod.POST,
        new HttpEntity<>(Map.of(
            "name", "후드 차콜", "brandId", brand.id(), "categoryId", category.id(),
            "basePriceKrw", 49000,
            "images", List.of(Map.of("url", "https://cdn.x/1.jpg", "sort", 0, "kind", "MAIN")),
            "options", List.of(Map.of("name", "Size", "value", "M",
                                      "extraPriceKrw", 0, "stock", 30))
        ), headers(auth)), Map.class);
    assertThat(createRes.getStatusCode().value()).isEqualTo(201);
    var productId = (String) ((Map<?, ?>) createRes.getBody().get("data")).get("productId");

    // GET
    var getRes = rest.exchange("/api/v1/seller/products/" + productId,
        HttpMethod.GET, new HttpEntity<>(headers(auth)), Map.class);
    assertThat(getRes.getStatusCode().value()).isEqualTo(200);

    // DELETE (soft)
    var delRes = rest.exchange("/api/v1/seller/products/" + productId,
        HttpMethod.DELETE, new HttpEntity<>(headers(auth)), Void.class);
    assertThat(delRes.getStatusCode().value()).isEqualTo(204);
}

@Test
void seller_cannot_modify_other_sellers_product() {
    var a = userFixture.createActiveSeller("a@x.com");
    var b = userFixture.createActiveSeller("b@x.com");
    var p = productFixture.createOwnedBy(a);

    var res = rest.exchange("/api/v1/seller/products/" + p.id(),
        HttpMethod.PATCH,
        new HttpEntity<>(Map.of("name", "hack"),
                         headers("Bearer " + tokens.access(b))),
        Map.class);
    assertThat(res.getStatusCode().value()).isEqualTo(403);
}
```

---

## 8. 운영 체크리스트

- [ ] `@Version` 으로 낙관 락 — 동시 수정 안전
- [ ] `@EntityGraph` / fetch join 으로 N+1 회피
- [ ] 카테고리 / 브랜드 변경 시 검색 색인 동기화 이벤트 발행
- [ ] 이미지 URL 은 HTTPS + 화이트리스트 CDN 도메인만
- [ ] 상품 수정 / 삭제 시 **변경 사항을 audit log** (관리자 화면)
- [ ] DRAFT 상품은 검색 / 공개 API 에 노출 X
- [ ] 옵션 100개 / 이미지 30개 한도 → 사용자에게 친절한 422
- [ ] soft delete 후에도 주문 / 장바구니 / 결제 이력에서 참조 가능
- [ ] price 변경 이력 보관 (감사 / 분쟁 대비) — `product_price_history` 별도 테이블 추천
- [ ] 셀러 본인 상품만 보이는 화면 + 관리자 화면 분리

---

## 9. 함정 모음

### 함정 1 — 도메인 객체를 JPA 와 합쳤는데 `@OneToMany` cascade 미설정
부모 저장 시 자식 안 저장. `cascade = ALL, orphanRemoval = true`.

### 함정 2 — `removeIf` 로 자식 제거했는데 DB 미반영
JPA 가 detached 상태로 두고 SQL 안 보냄. `orphanRemoval = true` 필수.

### 함정 3 — 옵션 unique 제약을 도메인만 검증
race condition. **DB `UNIQUE (product_id, name, value)`** 가 진실의 원천.

### 함정 4 — 카테고리 변경 시 자식 카테고리 처리 누락
부모만 바꾸고 path 안 갱신 → 카테고리 트리 쿼리 깨짐. **트리거** 또는 application 에서 일관 갱신.

### 함정 5 — 이미지 URL 신뢰
사용자 입력 URL 그대로 저장 = SSRF / phishing. **CDN 도메인 화이트리스트** + 가능하면 직접 업로드 후 우리 CDN URL.

### 함정 6 — `description` 의 HTML 그대로 저장
저장은 OK, **렌더 시 escape** + 관리자 화면은 sanitize (OWASP Java HTML Sanitizer).

### 함정 7 — `version` 컬럼 없이 동시 수정
A 가 가격 변경 + B 가 카테고리 변경 동시 → 한쪽 사라짐. `@Version` 필수.

### 함정 8 — soft delete 후에도 검색에 노출
DELETED status 를 검색 쿼리에서 빼는 것 잊음. **Repository 메서드 기본이 active only**.

### 함정 9 — 옵션 재고 차감을 도메인에서 끝
주문 동시성 처리는 [[order-stock]] 에서. **CRUD 만 다루는 이 레시피의 범위 밖**.

### 함정 10 — 권한 체크를 Controller 만
서비스 직접 호출 (예: 백그라운드 job) 시 우회. **도메인 (`product.isOwnedBy`) + Service** 에 항상 체크.

---

## 10. 다음 레시피

- [[product-search]] (예정) — 검색 / 필터 / 정렬 / 페이지네이션
- [[order-stock]] (예정) — 주문 + 재고 동시성 처리
- [[file-upload-s3]] (예정) — 이미지 직접 업로드 (presigned URL)
- review (예정) — 리뷰 + 평점 집계
- inventory-history (예정) — 재고 변동 이력
- price-history (예정) — 가격 변경 이력

---

## 11. 관련

- [[signup]] · [[login-jwt]] — 셀러 가입 / 로그인
- [[../../../../database/postgresql/security|↗ PG 보안]]
- [[api-design|↑ api-design hub]]
