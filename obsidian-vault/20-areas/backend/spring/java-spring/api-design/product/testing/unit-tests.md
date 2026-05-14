---
title: "Unit tests — 도메인 / aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:54:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - testing
  - unit
---

# Unit tests — 도메인 / aggregate

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ hub]]**

---

## 1. Payment aggregate

```java
@DisplayName("Payment 도메인")
class PaymentTest {

    @Test void initiate_생성시_READY() {
        var p = Payment.initiate(
            PaymentId.next(), OrderId.next(), UserId.next(),
            Money.krw(50000), PaymentMethod.CARD, "TOSS",
            Instant.now());
        assertThat(p.status()).isEqualTo(PaymentStatus.READY);
    }

    @Test void approve_idempotent() {
        var p = donePayment(50000);
        var beforeStatus = p.status();
        p.approve(Instant.now());
        assertThat(p.status()).isEqualTo(beforeStatus);     // no change
    }

    @Test void cancel_amount_exceeding_remaining_throws() {
        var p = donePayment(50000);
        p.cancel(Money.krw(30000), "partial", UserId.next(), Instant.now());
        assertThatThrownBy(() ->
            p.cancel(Money.krw(25000), "over", UserId.next(), Instant.now())
        ).isInstanceOf(IllegalArgumentException.class);
    }

    @Test void partialCancel_then_remaining() {
        var p = donePayment(50000);
        p.cancel(Money.krw(30000), "part1", UserId.next(), Instant.now());
        assertThat(p.status()).isEqualTo(PaymentStatus.PARTIAL_CANCELED);
        assertThat(p.refundableAmount()).isEqualTo(Money.krw(20000));

        p.cancel(Money.krw(20000), "part2", UserId.next(), Instant.now());
        assertThat(p.status()).isEqualTo(PaymentStatus.CANCELED);
    }
}
```

## 2. DigitalDelivery aggregate

```java
@Test void download_max_초과_throws() {
    var d = readyDelivery(/* max */ 5);
    for (int i = 0; i < 5; i++) d.download(Instant.now());
    assertThatThrownBy(() -> d.download(Instant.now()))
        .hasMessageContaining("download limit");
}

@Test void revoke_후_download_throws() {
    var d = readyDelivery(5);
    d.revoke("REFUND", Instant.now());
    assertThatThrownBy(() -> d.download(Instant.now()))
        .hasMessageContaining("revoked");
}
```

## 3. WatermarkInfo

```java
@Test void hash_decompose_consistent() {
    var info1 = WatermarkInfo.of(uid, oid, "a@b.com", t);
    var info2 = WatermarkInfo.of(uid, oid, "a@b.com", t);
    assertThat(info1.hash()).isEqualTo(info2.hash());
}

@Test void emailMasked_first_char_only() {
    var info = WatermarkInfo.of(uid, oid, "alice@example.com", t);
    assertThat(info.emailMasked()).isEqualTo("a***@example.com");
}
```

## 4. Money VO

```java
@Test void add_currency_mismatch_throws() {
    assertThatThrownBy(() -> Money.krw(100).add(new Money(BigDecimal.ONE, Currency.USD)))
        .hasMessageContaining("currency mismatch");
}
```

---

## 5. 관련

- [[testing|↑ hub]]
- [[../domain-model/payment-aggregate]]
- [[../domain-model/digital-delivery-aggregate]]
