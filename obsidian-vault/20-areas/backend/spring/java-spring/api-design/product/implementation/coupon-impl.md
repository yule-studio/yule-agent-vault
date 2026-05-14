---
title: "쿠폰 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - coupon
---

# 쿠폰 구현

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. Service

```java
@Service
@RequiredArgsConstructor
public class CouponService {

    private final CouponRepository coupons;
    private final CouponRedemptionRepository redemptions;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public CouponRedemption applyAndRecord(String code, UserId user, List<OrderItem> items) {
        // 1. 쿠폰 조회
        var coupon = coupons.findByCode(code).orElseThrow();
        if (!coupon.active() || coupon.expiresAt().isBefore(clock.now()))
            throw new CouponExpiredException();

        // 2. usage_count 차감 (atomic)
        int updated = coupons.incrementUsage(coupon.id(), coupon.usageLimit());
        if (updated == 0) throw new CouponSoldOutException();

        // 3. per_user_limit 검증
        int userUsed = redemptions.countByUserAndCoupon(user, coupon.id());
        if (userUsed >= coupon.perUserLimit())
            throw new CouponPerUserLimitException();

        // 4. 할인 계산
        var subtotal = items.stream()
            .map(i -> i.unitPrice().multiply(i.quantity()))
            .reduce(Money::add).orElseThrow();

        Money discount = switch (coupon.discountType()) {
            case FIXED_AMOUNT -> Money.krw(coupon.discountValue().longValue());
            case PERCENTAGE -> {
                var cap = coupon.maxDiscount() != null
                    ? Money.krw(coupon.maxDiscount().longValue())
                    : null;
                var raw = subtotal.multiply(coupon.discountValue());
                yield cap != null && raw.amountMinorUnit() > cap.amountMinorUnit()
                    ? cap : raw;
            }
            case FREE_SHIPPING -> Money.krw(0);   // shipping calc 별도
            case BUY_X_GET_Y -> calculateBxgy(items, coupon);
        };

        // 5. redemption row
        var redemption = new CouponRedemption(
            ids.next(), coupon.id(), user, /* orderId placeholder */, discount, clock.now());
        redemptions.save(redemption);
        return redemption;
    }

    public void restore(CouponRedemption redemption) {
        coupons.decrementUsage(redemption.couponId());
        redemption.markRestored(clock.now());
        redemptions.save(redemption);
    }
}
```

---

## 2. 함정

### 함정 1 — usage_count race
UPDATE WHERE usage < limit + version.

### 함정 2 — per_user 검증 없음
무한 사용.
→ COUNT.

### 함정 3 — PCT max cap 없음
1억 할인.
→ maxDiscount.

### 함정 4 — 만료 검증 없음
→ expiresAt.

### 함정 5 — 환불 시 복원 정책 명확 X
→ 본 vault 기본 X (정책 선택).

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/pricing-strategy]]
- [[../database/coupons-table]]
