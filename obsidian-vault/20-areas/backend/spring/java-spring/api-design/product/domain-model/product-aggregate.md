---
title: "Product Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:14:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - aggregate
---

# Product Aggregate

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. Aggregate 구조

```
Product (root)
├── List<ProductOption>          ← entity (axis)
├── List<Sku>                     ← entity (조합)
└── List<ProductImage>            ← entity
```

---

## 2. Java 코드

```java
public final class Product {
    private final ProductId id;
    private final UserId sellerId;
    private String name;
    private String description;
    private final ProductType type;            // PHYSICAL/DIGITAL/BOOK
    private ProductStatus status;
    private Money basePrice;
    private final boolean taxable;
    private final List<ProductOption> options = new ArrayList<>();
    private final List<Sku> skus = new ArrayList<>();
    private final List<ProductImage> images = new ArrayList<>();
    private long version;

    public static Product draft(ProductId id, UserId sellerId, String name,
                                ProductType type, Money basePrice, boolean taxable) {
        return new Product(id, sellerId, name, type, basePrice, taxable);
    }

    public void activate() {
        if (skus.isEmpty()) throw new IllegalStateException("at least 1 SKU required");
        if (status != ProductStatus.DRAFT) throw new IllegalStateException();
        this.status = ProductStatus.ACTIVE;
    }

    public void markSoldOut() {
        if (status == ProductStatus.ACTIVE) this.status = ProductStatus.SOLD_OUT;
    }

    public void restock() {
        if (status == ProductStatus.SOLD_OUT) this.status = ProductStatus.ACTIVE;
    }

    public void discontinue() {
        this.status = ProductStatus.DISCONTINUED;
    }

    public Sku addSku(SkuId id, String code, Money price, List<OptionValueId> values) {
        var sku = new Sku(id, this.id, code, price, values);
        skus.add(sku);
        return sku;
    }

    public void changePrice(Money newPrice, UserId changedBy, String reason) {
        // audit by event
        events.add(new ProductPriceChanged(this.id, this.basePrice, newPrice, changedBy, reason));
        this.basePrice = newPrice;
    }

    public Money priceFor(SkuId skuId) {
        return skus.stream()
            .filter(s -> s.id().equals(skuId))
            .findFirst().map(Sku::price)
            .orElseThrow();
    }
}
```

---

## 3. 핵심 메서드

| 메서드 | 책임 |
| --- | --- |
| `draft(...)` | 신규 → DRAFT |
| `activate()` | SKU 1개 이상 → ACTIVE |
| `markSoldOut()` / `restock()` | 재고 0 ↔ 복귀 |
| `discontinue()` | 단종 (irreversible) |
| `addSku(...)` | 옵션 조합 SKU 추가 |
| `changePrice(...)` | 가격 변경 + audit event |

---

## 4. 불변식

- DRAFT → ACTIVE: SKU 1개 이상.
- DISCONTINUED 후 ACTIVE 복귀 X.
- type 변경 X (PHYSICAL ↔ DIGITAL).
- price 음수 X.

---

## 5. 관련

- [[domain-model|↑ hub]]
- [[../enums/product-status]]
- [[../enums/product-type]]
- [[../database/products-table]]
- [[../design-decisions/product-status-policy]]
- [[../design-decisions/option-strategy]]
