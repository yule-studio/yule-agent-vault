---
title: "가격 / 할인 / 쿠폰 / 적립금 정책"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - pricing
  - coupon
---

# 가격 / 할인 / 쿠폰 / 적립금 정책

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> 가격 변경 / 할인 / 쿠폰 / 적립금의 결합 / 최종 가격 계산 순서.

---

## 1. 본 vault 결정

```
최종 가격 = (정가 × 수량) - 상품할인 - 쿠폰할인 - 적립금사용 + 배송비
```

- **쿠폰 1개 + 적립금 동시 사용 가능** (한국 일반).
- 가격 변경 시 audit + 옛 가격 보존.
- 할인 후 amount 검증 (서버에서 재계산).

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 서버 재계산 (FE 신뢰 X)

- FE 가 보낸 amount = 변조 가능.
- 서버에서 product / coupon / point 재 계산 → 일치 검증.

### 2.2 안 하면

| 잘못 | 사고 |
| --- | --- |
| FE amount 신뢰 | 100만원 결제를 100원으로 변조 |
| 쿠폰 중복 사용 | 1회용 쿠폰 N번 |
| 적립금 음수 | 환불 시 사용한 적립금 보다 더 |
| 가격 변경 시 audit X | 추후 분쟁 시 증거 X |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **할인 stack 1개 + 적립금** ★ | 본 vault |
| 모든 할인 stack | 마케팅 강화 (회계 복잡) |
| best discount auto | 사용자 친화 |
| AB test 가격 | grocery / 큰 platform |

---

## 3. 계산 순서

```java
public Money calculate(Order o) {
    var subtotal = items.stream()
        .map(i -> i.unitPrice().multiply(i.quantity()))
        .reduce(Money::add).orElse(Money.zero());

    var afterProductDiscount = subtotal.subtract(productDiscount);
    var afterCoupon = afterProductDiscount.subtract(coupon.calc(afterProductDiscount));
    var afterPoint = afterCoupon.subtract(pointUsed);
    var withShipping = afterPoint.add(shippingFee);
    return withShipping;
}
```

자세히: [[../implementation/coupon-impl]].

---

## 4. 쿠폰 모델

| Type | 적용 | 예 |
| --- | --- | --- |
| `FIXED_AMOUNT` | 5000원 할인 | `-5000` |
| `PERCENTAGE` | 10% 할인 (max 5000) | `-min(price*0.1, 5000)` |
| `FREE_SHIPPING` | 배송비 0 | `shippingFee = 0` |
| `BUY_X_GET_Y` | 2+1 | 복잡 — 별도 계산 |

---

## 5. 함정

### 함정 1 — FE amount 신뢰
변조 → 0원 결제.
→ 서버 재계산 + 비교.

### 함정 2 — 쿠폰 중복 사용
DB UPDATE WHERE used = false 없음 → race.
→ UNIQUE constraint + Optimistic.

### 함정 3 — 적립금 음수 (사용액 > 잔액)
중복 사용 / 환불 시 race.
→ point.deduct(amount) 검증.

### 함정 4 — 환불 시 적립금 안 복원
사용자 부당 이득 / 손실 (정책).

### 함정 5 — 쿠폰 만료 검증 X
만료된 쿠폰 사용.
→ expired_at 검증.

자세히: [[../pitfalls/payment-pitfalls]].

---

## 6. 다른 컨텍스트

### 6.1 글로벌 (Stripe Coupons)
Stripe API 의 coupon / promotion code 사용.

### 6.2 멤버십 (스타벅스)
적립금 = 카드 별 + 멤버십 등급 별 차등.

### 6.3 마켓플레이스
vendor 별 쿠폰 + 사이트 쿠폰 stack 가능.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[../database/coupons-table]]
- [[../implementation/coupon-impl]]
- [[refund-policy]] — 환불 시 적립금 / 쿠폰
- [[tax-strategy]]
