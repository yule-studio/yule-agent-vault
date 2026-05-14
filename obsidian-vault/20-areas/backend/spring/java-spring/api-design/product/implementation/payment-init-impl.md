---
title: "결제 init 구현 — PG SDK redirect"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:22:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - payment
---

# 결제 init 구현 — PG SDK redirect

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. 흐름

```
1. FE 가 /orders 로 order 생성 → orderId / amount
2. FE 가 PG SDK 호출 (orderId, amount, successUrl)
3. PG 가 결제창 open (iframe / redirect)
4. 사용자 결제
5. PG 가 successUrl 로 redirect (paymentKey 포함)
6. FE 가 /payments/confirm 호출
```

---

## 2. payments init API

본 vault 는 **별도 init API 없음** — order 생성 시 payment 도 READY 상태 INSERT.

```java
@Transactional
public CheckoutResult checkout(CheckoutCommand cmd, UserId buyer) {
    var order = ... // 위 [[checkout-impl]] 참고
    orders.save(order);

    var payment = Payment.initiate(
        PaymentId.next(),
        order.id(), buyer,
        order.totalAmount(),
        PaymentMethod.CARD,        // FE 가 변경
        "TOSS",                    // 본 vault 기본
        clock.now());
    payments.save(payment);

    return new CheckoutResult(order.id(), order.orderNumber(),
                              order.totalAmount(), payment.id());
}
```

---

## 3. FE SDK 통합 (Toss)

```javascript
import { loadTossPayments } from '@tosspayments/payment-sdk';

const toss = await loadTossPayments(CLIENT_KEY);

await toss.requestPayment('CARD', {
    amount: order.totalAmount,
    orderId: order.orderNumber,        // PG 는 사용자 친화 number 사용
    orderName: order.itemSummary,
    successUrl: `${location.origin}/payments/success`,
    failUrl: `${location.origin}/payments/fail`,
});
```

→ 서버 호출 없음 (FE 가 직접 PG SDK 호출).

---

## 4. successUrl 처리

```javascript
// /payments/success?paymentKey=...&orderId=...&amount=...
const params = new URLSearchParams(location.search);

const idempotencyKey = crypto.randomUUID();
const result = await fetch('/api/v1/payments/confirm', {
    method: 'POST',
    headers: {
        'Idempotency-Key': idempotencyKey,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        paymentKey: params.get('paymentKey'),
        orderId: params.get('orderId'),
        amount: Number(params.get('amount')),
    }),
});
```

자세히: [[payment-confirm-impl]].

---

## 5. 함정

### 함정 1 — orderId 가 ULID 사용
PG 에 보낸 orderId 와 서버 의 order.id 가 다름 — 매핑 어려움.
→ `orderNumber` (사용자 친화) 사용 + 서버에서 매핑.

### 함정 2 — payment row 안 만들고 confirm 만 INSERT
race + audit 부족.
→ order 와 동시 INSERT (READY 상태).

### 함정 3 — clientKey hardcode
public key 라도 환경별 분리.
→ env.

---

## 6. 관련

- [[implementation|↑ hub]]
- [[payment-confirm-impl]]
- [[../design-decisions/payment-flow]]
- [[../design-decisions/pg-selection]]
