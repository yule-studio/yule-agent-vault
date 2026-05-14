---
title: "order_items 테이블 — snapshot 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:48:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - order
---

# order_items 테이블 — snapshot 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 주문의 각 line item — **상품 정보 snapshot** (가격 / 옵션 변경에 영향 X).

---

## 1. Schema

```sql
-- V31__create_order_items.sql
CREATE TABLE order_items (
    id              CHAR(26) PRIMARY KEY,
    order_id        CHAR(26) NOT NULL REFERENCES orders(id),
    product_id      CHAR(26) NOT NULL,                  -- snapshot, no FK
    sku_id          CHAR(26) NOT NULL,                  -- snapshot
    product_name    VARCHAR(200) NOT NULL,              -- snapshot
    product_type    VARCHAR(20) NOT NULL,               -- snapshot (PHYSICAL/DIGITAL/BOOK)
    sku_code        VARCHAR(50) NOT NULL,               -- snapshot
    option_values   JSONB NOT NULL,                     -- snapshot { "size": "L", "color": "red" }

    unit_price_amount NUMERIC(15, 4) NOT NULL,          -- snapshot
    unit_price_currency CHAR(3) NOT NULL,
    quantity        INTEGER NOT NULL CHECK (quantity > 0),

    item_subtotal_amount NUMERIC(15, 4) NOT NULL,       -- unit_price × quantity
    item_discount_amount NUMERIC(15, 4) NOT NULL DEFAULT 0,
    item_tax_amount NUMERIC(15, 4) NOT NULL DEFAULT 0,
    item_total_amount NUMERIC(15, 4) NOT NULL,

    refunded_amount NUMERIC(15, 4) NOT NULL DEFAULT 0,
    fulfilled       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_order_items_order ON order_items (order_id);
CREATE INDEX ix_order_items_product ON order_items (product_id);     -- 통계
```

---

## 2. 왜 snapshot

### 2.1 왜 product 정보 복사

- 가격 변경 → 옛 주문 가격도 변경 = X.
- 상품 단종 → 옛 주문 데이터 사라짐 = X.
- snapshot = 주문 시점의 fact.

### 2.2 왜 FK 없음 (product_id 만)

- product soft delete 후 FK 무효.
- snapshot = "이 주문은 이 product 였다" — link 만, 데이터는 자기 소유.

### 2.3 왜 option_values JSONB

- 옵션 변경 / 단종 후에도 표시 가능.
- 영수증 / 환불 / CS 모두 사용.

### 2.4 `refunded_amount`

- 부분 환불 추적 (item 별).
- `item_total - refunded` = 잔여 환불 가능.

### 2.5 `fulfilled BOOLEAN`

- item 별 배송 완료 (실물 1개 + 디지털 1개 — 디지털 먼저 fulfill 가능).

---

## 3. 함정

### 함정 1 — product FK 사용
soft delete 시 무효.
→ snapshot.

### 함정 2 — snapshot 안 함
가격 변경 시 옛 주문도 변경.
→ snapshot.

### 함정 3 — refunded_amount 안 추적 (order 만)
부분 환불 시 어떤 item 인지 X.
→ item 별 refunded.

### 함정 4 — fulfilled order 단위
실물 / 디지털 혼합 시 추적 X.
→ item 별 fulfilled.

---

## 4. 관련

- [[database|↑ hub]]
- [[orders-table]]
- [[refunds-table]]
- [[../design-decisions/refund-policy]]
