---
title: "통화 전략 — KRW only → 다국적 (Money VO)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - currency
  - i18n
---

# 통화 전략 — KRW only → 다국적 (Money VO)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> KRW only (1단계) — 그러나 **Money VO 추상화** 로 글로벌 대비.

---

## 1. 본 vault 결정

- **F0~F8**: KRW only (정수 long).
- **Money VO**: 항상 `Money(BigDecimal, Currency)` — Currency 가 항상 KRW.
- **F12+**: 글로벌 — `Currency` enum 확장, 환율 / 멀티 통화 정산.

---

## 2. 왜 Money VO (KRW 만 인데도)

### 2.1 왜 필요

- 통화 무시 → 다국적 진입 시 6개월 재작업.
- Long krw 가 코드 전반에 흩어짐 — 의미 추적 어려움.

### 2.2 안 하면

| 잘못 | 사고 |
| --- | --- |
| `long price` | 다국적 진입 시 모든 코드 변경 |
| 덧셈 / 비교 가 raw long | currency 불일치 (KRW + USD = 의미 X) 컴파일러 못 막음 |
| FE 의 amount 가 string vs number | 화폐 단위 (minor unit) 혼동 (KRW 100 = 100원, USD 100 = $1.00) |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **Money(BigDecimal, Currency)** ★ | 본 vault — i18n 대비 |
| long krw | MVP — but 진입 어려움 |
| Joda Money / javax.money | 표준 lib |
| Stripe-style minor unit | int (cents) |

---

## 3. Money VO

```java
public record Money(BigDecimal amount, Currency currency) {
    public static Money krw(long won) {
        return new Money(BigDecimal.valueOf(won), Currency.KRW);
    }
    public static Money zero(Currency c) {
        return new Money(BigDecimal.ZERO, c);
    }
    public Money add(Money other) {
        require(currency.equals(other.currency), "currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
    public Money subtract(Money other) { /* ... */ }
    public Money multiply(int qty) {
        return new Money(amount.multiply(BigDecimal.valueOf(qty)), currency);
    }
    public long amountMinorUnit() {
        return currency.toMinorUnit(amount);
    }
}

public enum Currency {
    KRW(0), USD(2), EUR(2), JPY(0);
    private final int scale;
    public long toMinorUnit(BigDecimal amt) {
        return amt.movePointRight(scale).longValueExact();
    }
}
```

자세히: [[../domain-model/value-objects]] · [[../enums/currency]].

---

## 4. DB

```sql
order_items (
    unit_price_amount NUMERIC(15, 4) NOT NULL,
    unit_price_currency CHAR(3) NOT NULL DEFAULT 'KRW'
);
```

→ `_krw` suffix 안 씀. minor unit 컨버전은 코드.

---

## 5. 함정

### 함정 1 — long krw 사용
다국적 진입 시 전체 코드 변경.
→ Money VO.

### 함정 2 — float / double
부동소수점 — 1.1 + 2.2 != 3.3.
→ BigDecimal.

### 함정 3 — currency mismatch 미검증
KRW + USD = 의미 X.
→ Money.add 에서 검증.

### 함정 4 — minor unit 혼동
"100 KRW" = 100원, "100 USD" = $1.00.
→ `amountMinorUnit()` 명확.

---

## 6. 다른 컨텍스트

### 6.1 글로벌 SaaS (Stripe)
multi-currency 결제 + 환율 자동 + Stripe-Tax.

### 6.2 크로스보더 (Aliexpress)
표시 currency vs 결제 currency 분리 — 환율 lock 시점 결정.

### 6.3 가상화폐 (NFT)
ETH / USDC — Currency enum 확장 + decimal 18.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[../domain-model/value-objects]]
- [[../enums/currency]]
- [[tax-strategy]]
- [[pricing-strategy]]
