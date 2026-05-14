---
title: "products + product_skus 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
---

# products + product_skus 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

---

## 1. Schema

```sql
-- V20__create_products.sql
CREATE TABLE products (
    id              CHAR(26) PRIMARY KEY,
    seller_id       CHAR(26) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    type            VARCHAR(20) NOT NULL,        -- PHYSICAL / DIGITAL / BOOK
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    base_price_amount NUMERIC(15, 4) NOT NULL CHECK (base_price_amount >= 0),
    base_price_currency CHAR(3) NOT NULL DEFAULT 'KRW',
    tax_rate        NUMERIC(5, 4) NOT NULL DEFAULT 0.10,  -- 0 = 면세 (도서)
    is_taxable      BOOLEAN NOT NULL DEFAULT TRUE,
    category_id     CHAR(26),
    metadata        JSONB,                       -- 자유 확장 (저자, ISBN, ...)
    version         BIGINT NOT NULL DEFAULT 0,
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_product_status CHECK
        (status IN ('DRAFT', 'ACTIVE', 'SOLD_OUT', 'DISCONTINUED')),
    CONSTRAINT chk_product_type CHECK
        (type IN ('PHYSICAL', 'DIGITAL', 'BOOK'))
);

CREATE INDEX ix_products_status ON products (status) WHERE deleted_at IS NULL;
CREATE INDEX ix_products_category ON products (category_id) WHERE status = 'ACTIVE';
CREATE INDEX ix_products_seller ON products (seller_id);
CREATE INDEX ix_products_search ON products USING gin (to_tsvector('korean', name || ' ' || COALESCE(description, '')));
```

```sql
-- V23__create_product_skus.sql
CREATE TABLE product_skus (
    id              CHAR(26) PRIMARY KEY,
    product_id      CHAR(26) NOT NULL REFERENCES products(id),
    sku_code        VARCHAR(50) NOT NULL UNIQUE,
    price_amount    NUMERIC(15, 4) NOT NULL,
    price_currency  CHAR(3) NOT NULL DEFAULT 'KRW',
    barcode         VARCHAR(50),
    weight_g        INTEGER,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_skus_product ON product_skus (product_id);

CREATE TABLE product_sku_options (
    sku_id          CHAR(26) NOT NULL REFERENCES product_skus(id),
    option_value_id CHAR(26) NOT NULL REFERENCES product_option_values(id),
    PRIMARY KEY (sku_id, option_value_id)
);
```

---

## 2. 컬럼 "왜"

### 2.1 `type` (PHYSICAL / DIGITAL / BOOK)

- DIGITAL vs BOOK 의 차이: BOOK 은 면세 + 마스킹 워크플로 trigger.
- 결제 흐름은 같지만 fulfillment 가 다름.

### 2.2 `is_taxable` + `tax_rate`

- 책 = 면세 (`is_taxable=false`).
- 일반 상품 = 0.10.
- 변경 시 audit (price_history 와 별도 — 면세 변경은 드물지만).

### 2.3 `metadata JSONB`

- type 별 동적 필드 (BOOK: ISBN/author / DIGITAL: assetId / PHYSICAL: weight_g).
- 너무 검색이 잦으면 별도 컬럼 승격.

### 2.4 `version BIGINT`

- Optimistic lock — 가격 / 상태 동시 변경 race.

### 2.5 `deleted_at` (soft delete)

- 옛 주문의 정산 / 환불 시 product 정보 필요.
- 영구 삭제 X.

---

## 3. SKU 패턴

- 옵션 없으면 SKU 1개 (`is_default=true`).
- 옵션 있으면 카르테시안 곱.
- `sku_code` UNIQUE — 외부 시스템 (창고 / 회계) 통합.

자세히: [[../design-decisions/option-strategy]].

---

## 4. 함정

### 함정 1 — products.price 가 SKU 모름
옵션 별 가격 차이 표현 X.
→ SKU price 가 권위.

### 함정 2 — hard delete
옛 주문의 환불 시 product 정보 사라짐.
→ soft delete.

### 함정 3 — name FTS index 한국어 안 함
검색 정확도 ↓.
→ to_tsvector('korean', ...).

### 함정 4 — type 변경 허용
DIGITAL → PHYSICAL 변경 시 fulfillment 가 무엇이었는지 모름.
→ type 변경 금지 (admin 도 X).

---

## 5. 관련

- [[database|↑ hub]]
- [[product-options-table]]
- [[inventory-table]]
- [[../design-decisions/option-strategy]]
- [[../design-decisions/product-status-policy]]
- [[../enums/product-type]]
