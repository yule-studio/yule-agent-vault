---
title: "Currency enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:48:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - enum
---

# Currency enum

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ hub]]**

---

## 1. 값

```java
public enum Currency {
    KRW(0, "₩"),
    USD(2, "$"),
    EUR(2, "€"),
    JPY(0, "¥");

    private final int scale;     // BigDecimal scale (minor unit)
    private final String symbol;

    public long toMinorUnit(BigDecimal amt) {
        return amt.movePointRight(scale).longValueExact();
    }
}
```

## 2. F0~F8 = KRW only

- DB 컬럼: `currency CHAR(3) NOT NULL DEFAULT 'KRW'`.
- 단일 값 — 코드 단순.
- but Money VO 는 이미 currency 인자 받음 → F12+ 다국적 진입 시 코드 변경 X.

## 3. F12+ 다국적

- USD / EUR / JPY 추가.
- 환율 lock 시점: 결제 시.
- Stripe-Tax / Stripe-Tax-equivalent 통합.

자세히: [[../design-decisions/currency-strategy]].

## 4. 관련

- [[enums|↑ hub]]
- [[../domain-model/value-objects]]
- [[../design-decisions/currency-strategy]]
