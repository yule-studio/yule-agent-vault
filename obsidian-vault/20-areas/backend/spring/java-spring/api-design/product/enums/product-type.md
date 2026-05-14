---
title: "ProductType enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:43:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - enum
---

# ProductType enum

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ hub]]**

---

## 1. 값

```java
public enum ProductType {
    PHYSICAL,    // 실물 — 택배 배송
    DIGITAL,     // 디지털 — 다운로드 (영상 / SW)
    BOOK;        // 책 (PDF) — 면세 + 워터마크
}
```

## 2. type 별 정책

| Type | 면세 | 배송 | 환불 | 워터마크 |
| --- | --- | --- | --- | --- |
| PHYSICAL | X (10% VAT) | 택배 API | 7일 단순변심 | X |
| DIGITAL | X | 다운로드 | 1회 재생 / 사용 전만 | 옵션 |
| BOOK | **O (면세)** | 다운로드 (마스킹 PDF) | **다운로드 전만** | **필수** |

## 3. type 불변

- 등록 후 type 변경 X — fulfillment 흐름이 다름.
- 책을 강의로 변경 시 새 상품 등록.

## 4. 관련

- [[enums|↑ hub]]
- [[../design-decisions/digital-delivery-policy]]
- [[../design-decisions/physical-delivery-policy]]
- [[../design-decisions/refund-policy]]
- [[../design-decisions/tax-strategy]]
