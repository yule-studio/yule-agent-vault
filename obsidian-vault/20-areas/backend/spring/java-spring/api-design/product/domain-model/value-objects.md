---
title: "Value Objects — Money / Ids / Address"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - value-object
---

# Value Objects — Money / Ids / Address

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. ID VOs (ULID-backed)

```java
public record ProductId(String value) {
    public ProductId { require(value.length() == 26, "ULID required"); }
    public static ProductId next() { return new ProductId(Ulid.fast().toString()); }
}

public record SkuId(String value) { ... }
public record OrderId(String value) { ... }
public record OrderItemId(String value) { ... }
public record PaymentId(String value) { ... }
public record DeliveryId(String value) { ... }
public record AssetId(String value) { ... }
public record UserId(String value) { ... }
```

### 1.1 왜 type-safe Id (vs raw String)

- `String orderId` vs `String paymentId` 컴파일러 못 막음 → 잘못된 인자 전달.
- VO = type 안전.

---

## 2. Money

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        require(amount.scale() <= currency.scale(), "scale exceeds currency");
    }
    public static Money krw(long won) { ... }
    public static Money zero(Currency c) { ... }
    public Money add(Money o) { ... }
    public Money subtract(Money o) { ... }
    public Money multiply(int q) { ... }
    public long amountMinorUnit() { ... }
}
```

자세히: [[../design-decisions/currency-strategy]].

---

## 3. Address (encrypted)

```java
public record Address(
    String recipientName,
    String phone,
    String zipCode,
    String addressLine1,
    String addressLine2,
    String city,
    String country) {
}

// DB 저장 시 pgcrypto encrypt
@Converter
public class AddressEncryptedConverter implements AttributeConverter<Address, byte[]> {
    public byte[] convertToDatabaseColumn(Address a) { return crypto.encrypt(json(a)); }
    public Address convertToEntityAttribute(byte[] b) { return parse(crypto.decrypt(b)); }
}
```

자세히: [[../security/pii-encryption]].

---

## 4. WatermarkInfo

```java
public record WatermarkInfo(
    UserId userId,
    OrderId orderId,
    String emailMasked,
    Instant purchasedAt,
    String hash) {

    public static WatermarkInfo of(UserId u, OrderId o, String email, Instant t) {
        var masked = maskEmail(email);
        var hash = sha256(u.value() + o.value() + masked + t.toString());
        return new WatermarkInfo(u, o, masked, t, hash);
    }

    private static String maskEmail(String e) {
        // a***@gmail.com
        var at = e.indexOf('@');
        return e.charAt(0) + "***" + e.substring(at);
    }
}
```

자세히: [[../security/digital-watermarking]].

---

## 5. 함정

### 함정 1 — raw String id
타입 안전 X.
→ record VO.

### 함정 2 — Money 의 BigDecimal scale 무시
KRW = scale 0, USD = scale 2 — 1.50 KRW 의미 X.
→ Currency.scale 검증.

### 함정 3 — Address 평문 DB 저장
PII 유출.
→ pgcrypto + AttributeConverter.

---

## 6. 관련

- [[domain-model|↑ hub]]
- [[../enums/currency]]
- [[../design-decisions/currency-strategy]]
- [[../security/pii-encryption]]
- [[../security/digital-watermarking]]
