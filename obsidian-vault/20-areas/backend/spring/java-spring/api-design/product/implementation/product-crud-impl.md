---
title: "상품 CRUD 구현 (admin)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:18:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
---

# 상품 CRUD 구현 (admin)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. API

```http
POST   /api/v1/admin/products
GET    /api/v1/admin/products/{id}
PATCH  /api/v1/admin/products/{id}
DELETE /api/v1/admin/products/{id}                      (soft)
POST   /api/v1/admin/products/{id}/activate
POST   /api/v1/admin/products/{id}/discontinue
POST   /api/v1/admin/products/{id}/skus
PATCH  /api/v1/admin/products/{id}/skus/{skuId}/price

GET    /api/v1/products?category=...&q=...              (public 카탈로그)
GET    /api/v1/products/{id}                            (public)
```

---

## 2. Application service

```java
@Service
@RequiredArgsConstructor
public class ProductCommandService {

    private final ProductRepository products;
    private final IdGenerator ids;
    private final HtmlSanitizer sanitizer;
    private final Clock clock;

    @Transactional
    public ProductId create(CreateProductCommand cmd, UserId actor) {
        var id = ProductId.next();
        var product = Product.draft(id, actor,
            cmd.name(),
            cmd.type(),
            Money.krw(cmd.basePrice()),
            cmd.taxable());

        // XSS
        product.setDescription(sanitizer.sanitize(cmd.description()));

        products.save(product);
        return id;
    }

    @Transactional
    public void activate(ProductId id) {
        var p = products.findById(id).orElseThrow();
        p.activate();
        products.save(p);
    }

    @Transactional
    public void changePrice(ProductId id, long newWon, UserId actor, String reason) {
        var p = products.findById(id).orElseThrow();
        p.changePrice(Money.krw(newWon), actor, reason);
        products.save(p);
    }
}
```

---

## 3. XSS sanitization

```java
@Component
class HtmlSanitizer {
    private final PolicyFactory policy = new HtmlPolicyBuilder()
        .allowElements("p", "br", "ul", "li", "strong", "em", "h2", "h3")
        .allowUrlProtocols("https")
        .allowAttributes("href").onElements("a")
        .toFactory();

    public String sanitize(String html) {
        return policy.sanitize(html);
    }
}
```

자세히: [[../../board/security/xss-defense|↗ board xss-defense]].

---

## 4. 검증

| 항목 | 검증 |
| --- | --- |
| name | 1~200 char |
| basePrice | >= 0 |
| type 변경 | 등록 후 X |
| status 전이 | 도메인 메서드만 |

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../domain-model/product-aggregate]]
- [[../database/products-table]]
- [[../design-decisions/product-status-policy]]
