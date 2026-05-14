---
title: "실물 배송 구현 — 택배 API + tracking"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:36:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - shipping
---

# 실물 배송 구현 — 택배 API + tracking

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. Listener

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentApproved(PaymentApproved ev) {
    var order = orders.findById(ev.orderId()).orElseThrow();
    if (!order.hasPhysicalItem()) return;

    var shipment = Shipment.create(
        ids.next(), order.id(), order.shippingAddress(),
        clock.now());
    shipments.save(shipment);

    // 송장 발급 worker enqueue
}
```

---

## 2. Worker (송장 발급)

```java
@Scheduled(fixedDelay = 10_000)
public void issueInvoices() {
    var pending = shipments.findReadyForInvoice(20);
    for (var s : pending) issueOne(s);
}

@Transactional
public void issueOne(Shipment s) {
    var provider = router.resolve("CJ");
    var invoice = provider.issue(new ShippingCommand(s.address(), s.items()));
    s.setTrackingNumber(invoice.trackingNumber());
    s.markPickedUp(clock.now());
    shipments.save(s);

    notifications.send(s.userId(), "SHIPPING_STARTED",
        Map.of("trackingNumber", invoice.trackingNumber()));
}
```

---

## 3. Webhook (택배 status 변경)

```java
@PostMapping("/api/v1/webhooks/shipping/cj")
public ResponseEntity<Void> cjWebhook(@RequestBody CjEvent ev) {
    var s = shipments.findByTrackingNumber(ev.trackingNumber()).orElseThrow();
    switch (ev.status()) {
        case "IN_TRANSIT" -> s.markInTransit(clock.now());
        case "DELIVERED" -> {
            s.markDelivered(clock.now());
            var order = orders.findById(s.orderId()).orElseThrow();
            order.fulfill(clock.now());
            orders.save(order);
        }
    }
    shipments.save(s);
    return ResponseEntity.ok().build();
}
```

---

## 4. 함정

### 함정 1 — 송장 발급 동기 (결제 confirm 안)
PG 호출 timeout → DB 락.
→ AFTER_COMMIT + worker.

### 함정 2 — webhook 누락 시 fulfill X
일부 택배사 webhook 의 신뢰성 ↓.
→ polling 매일 (옛 IN_TRANSIT).

### 함정 3 — 옛 주소로 송장 발급
사용자가 주소 변경했는데 발급 직전.
→ 발급 직전 address snapshot.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/physical-delivery-policy]]
- [[../enums/delivery-status]]
