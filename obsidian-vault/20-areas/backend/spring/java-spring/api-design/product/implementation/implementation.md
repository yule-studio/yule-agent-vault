---
title: "product implementation hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
---

# product implementation hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

---

## 1. 영역

| 노트 | 영역 |
| --- | --- |
| [[product-crud-impl]] | F1 — 상품 CRUD |
| [[checkout-impl]] | F4 — 주문 생성 |
| [[payment-init-impl]] | F5 — PG init / redirect |
| [[payment-confirm-impl]] ★ | F5 — /confirm + 멱등 + Kafka outbox |
| [[payment-webhook-impl]] ★ | F5 — webhook 수신 |
| [[refund-impl]] | F8 — 환불 |
| [[digital-delivery-impl]] ★ | F6 — 워터마크 worker + GDrive |
| [[physical-delivery-impl]] | F7 — 택배 API |
| [[inventory-impl]] | F4 — 재고 차감 (Redis Lua) |
| [[coupon-impl]] | F4 — 쿠폰 |
| [[settlement-impl]] | F9 — 정산 |
| [[kafka-integration]] ★ | F10+ — Outbox + producer + consumer |

---

## 2. 관련

- [[../product|↑ hub]]
- [[../architecture]]
- [[../transactions]]
