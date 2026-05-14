---
title: "주문 생성 (checkout) 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - checkout
---

# 주문 생성 (checkout) 구현

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. API

```http
POST /api/v1/orders
{
  "items": [
    { "skuId": "sku_01HX...", "quantity": 2 }
  ],
  "couponCode": "WELCOME10",
  "pointUsed": 5000,
  "shippingAddress": { ... }
}

201 Created
{
  "orderId": "ord_01HX...",
  "orderNumber": "ORD-2026-A1B2C3",
  "totalAmount": 50000,
  "currency": "KRW"
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class CheckoutService {

    private final ProductRepository products;
    private final OrderRepository orders;
    private final InventoryService inventory;
    private final CouponService coupons;
    private final PointService points;
    private final ShippingFeeCalculator shippingCalc;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public CheckoutResult checkout(CheckoutCommand cmd, UserId buyer) {
        // 1. items snapshot
        var items = cmd.items().stream().map(i -> {
            var sku = products.findSkuById(i.skuId()).orElseThrow();
            return OrderItem.of(
                OrderItemId.next(),
                sku.productId(), sku.id(),
                sku.productName(),
                sku.productType(),
                sku.code(),
                sku.optionValues(),
                sku.price(),
                i.quantity()
            );
        }).toList();

        // 2. 재고 차감 (Redis Lua atomic)
        for (var item : items) {
            inventory.decrement(item.skuId(), item.quantity());
        }

        // 3. 쿠폰 + 적립금
        var coupon = cmd.couponCode() != null
            ? Optional.of(coupons.applyAndRecord(cmd.couponCode(), buyer, items))
            : Optional.<CouponRedemption>empty();
        points.deduct(buyer, Money.krw(cmd.pointUsed()));

        // 4. 배송비
        var shipping = shippingCalc.calculate(items, cmd.shippingAddress());

        // 5. order
        var order = Order.create(
            OrderId.next(), buyer, items,
            shipping, /* tax */ shippingCalc.tax(items),
            coupon, Money.krw(cmd.pointUsed()),
            cmd.shippingAddress(), clock.now());

        orders.save(order);
        return new CheckoutResult(order.id(), order.orderNumber(), order.totalAmount());
    }
}
```

---

## 3. 트랜잭션 경계

- 재고 차감 (Redis) = `@Transactional` 안에서 호출 — 실패 시 throw → DB rollback.
- 외부 호출 없음 (PG 는 별도 API — [[payment-confirm-impl]]).

---

## 4. 30분 timeout worker

```java
@Component
@RequiredArgsConstructor
public class OrderTimeoutWorker {
    private final OrderRepository orders;
    private final InventoryService inventory;
    private final Clock clock;

    @Scheduled(fixedDelay = 60_000)
    @SchedulerLock(name = "orderTimeoutWorker", lockAtMostFor = "5m")
    public void scan() {
        var expired = orders.findExpiredPending(Duration.ofMinutes(30), clock.now());
        for (var order : expired) {
            order.cancel("TIMEOUT", clock.now());
            for (var item : order.items())
                inventory.restore(item.skuId(), item.quantity());
            orders.save(order);
        }
    }
}
```

---

## 5. 함정

### 함정 1 — 재고 차감 트랜잭션 밖
DB rollback 후 Redis 차감 그대로 — 재고 손실.
→ 트랜잭션 안 + 실패 시 throw.

### 함정 2 — items snapshot 안 함
가격 변경 시 옛 주문 가격도 변경.
→ snapshot.

### 함정 3 — DRAFT / SOLD_OUT / DISCONTINUED 결제
overselling / 단종 상품 결제.
→ Product.status = ACTIVE 검증.

### 함정 4 — timeout 없음
재고 영구 잠김.
→ scanExpired.

---

## 6. 관련

- [[implementation|↑ hub]]
- [[../domain-model/order-aggregate]]
- [[../database/orders-table]]
- [[../design-decisions/inventory-strategy]]
- [[inventory-impl]]
- [[coupon-impl]]
