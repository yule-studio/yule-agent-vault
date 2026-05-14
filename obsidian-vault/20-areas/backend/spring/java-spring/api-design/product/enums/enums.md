---
title: "product enums hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - enum
---

# product enums hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

> 8개 enum. 각각의 design rationale + 상태 머신 + DB CHECK.

---

## 1. 목록

| Enum | 자세히 |
| --- | --- |
| ProductStatus | [[product-status]] |
| ProductType | [[product-type]] |
| OrderStatus | [[order-status]] |
| PaymentStatus | [[payment-status]] |
| PaymentMethod | [[payment-method]] |
| RefundStatus | [[refund-status]] |
| Currency | [[currency]] |
| DeliveryStatus | [[delivery-status]] |

---

## 2. 공통 패턴

- VARCHAR + DB CHECK 제약 (enum table X — overkill).
- application: Java `enum`.
- 상태 머신 mermaid 명시.
- 추가 / 변경 시 migration + audit.

---

## 3. 관련

- [[../product|↑ hub]]
- [[../domain-model/domain-model]]
- [[../database/database]]
