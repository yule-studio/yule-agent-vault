---
title: "orders 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - order
---

# orders 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

---

## 1. Schema

```sql
-- V30__create_orders.sql
CREATE TABLE orders (
    id              CHAR(26) PRIMARY KEY,
    order_number    VARCHAR(20) NOT NULL UNIQUE,        -- 사용자 친화 (예: ORD-2026-A1B2C3)
    buyer_id        CHAR(26) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    subtotal_amount NUMERIC(15, 4) NOT NULL,
    discount_amount NUMERIC(15, 4) NOT NULL DEFAULT 0,
    coupon_amount   NUMERIC(15, 4) NOT NULL DEFAULT 0,
    point_amount    NUMERIC(15, 4) NOT NULL DEFAULT 0,
    shipping_amount NUMERIC(15, 4) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(15, 4) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(15, 4) NOT NULL,            -- 최종 결제 금액
    currency        CHAR(3) NOT NULL DEFAULT 'KRW',
    coupon_id       CHAR(26),
    shipping_address_encrypted BYTEA,                    -- pgcrypto
    shipping_method VARCHAR(20),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    paid_at         TIMESTAMPTZ,
    fulfilled_at    TIMESTAMPTZ,
    canceled_at     TIMESTAMPTZ,
    cancel_reason   VARCHAR(500),
    version         BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_order_status CHECK
        (status IN ('PENDING', 'PAID', 'FULFILLED', 'CANCELED', 'REFUNDED'))
);

CREATE INDEX ix_orders_buyer_created ON orders (buyer_id, created_at DESC);
CREATE INDEX ix_orders_status_requested ON orders (status, requested_at);    -- timeout scan
CREATE INDEX ix_orders_pending_expired ON orders (requested_at) WHERE status = 'PENDING';
```

---

## 2. 컬럼 "왜"

### 2.1 `order_number` (vs id)

- ULID = 내부 / DB.
- order_number = 사용자 노출 (CS 통화 시 "ORD-2026-A1B2C3").
- UNIQUE — 검색 빠름.

### 2.2 amount 컬럼 split

- subtotal / discount / coupon / point / shipping / tax / total 분리.
- 회계 / 환불 시 각 column 차감.
- 합산 검증: `total = subtotal - discount - coupon - point + shipping + tax`.

### 2.3 `shipping_address_encrypted BYTEA`

- pgcrypto 암호화 (PII).
- key 는 KMS / vault.

### 2.4 partial index `pending_expired`

- 30분 timeout scan 빠르게.

---

## 3. 함정

### 함정 1 — order_number 가 id 자체
사용자가 ULID 봄 — UX ↓.
→ 별도 컬럼 + 친화 포맷.

### 함정 2 — amount 합계 컬럼 1개
환불 시 어떤 부분 차감인지 X.
→ split 컬럼.

### 함정 3 — 주소 평문
PII 유출 위험.
→ pgcrypto.

### 함정 4 — 옛 옵션 변경 시 order 변경
사용자가 옵션 변경 → 옛 주문도 변경.
→ snapshot (order_items 에 옵션 값 hardcode).

자세히: [[order-items-table]].

---

## 4. 관련

- [[database|↑ hub]]
- [[order-items-table]]
- [[payments-table]]
- [[../enums/order-status]]
- [[../security/pii-encryption]]
- [[../design-decisions/pricing-strategy]]
