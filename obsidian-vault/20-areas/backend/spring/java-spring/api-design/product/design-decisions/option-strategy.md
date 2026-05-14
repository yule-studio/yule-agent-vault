---
title: "옵션 / SKU 전략 — Variant 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - option
  - sku
---

# 옵션 / SKU 전략 — Variant 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> 옷 size (S/M/L) × color (red/blue) = 6 SKU. 옵션 모델링 방법.

---

## 1. 본 vault 결정

- 1 Product + N ProductOptions (axis 단위) + N SKU (조합).
- 재고 / 가격 = SKU 단위.

```
Product: "티셔츠"
├── ProductOption: "size" (S, M, L)
├── ProductOption: "color" (red, blue)
└── SKU (size + color 의 조합):
    ├── SKU-001: S + red,    stock=10, price=20000
    ├── SKU-002: S + blue,   stock=5,  price=20000
    ├── SKU-003: M + red,    stock=15, price=22000
    ├── SKU-004: M + blue,   stock=0,  price=22000  (옵션별 SOLD_OUT 가능)
    ├── SKU-005: L + red,    stock=8,  price=22000
    └── SKU-006: L + blue,   stock=12, price=22000
```

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 SKU 단위 재고

- 옵션 별 재고 분리 (red XL 만 SOLD_OUT 가능).
- 옵션 별 가격 차이 (L 사이즈 + 2000원).

### 2.2 안 하면

| 잘못 | 사고 |
| --- | --- |
| Product 단위 재고 | red 다 떨어졌는데 blue 도 SOLD_OUT 표시 |
| JSON 컬럼 옵션 | 검색 / 필터 어려움 + 재고 차감 race |
| 옵션 hardcode | 새 옵션 (sleeve length) 추가 시 schema 변경 |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **Product + Options + SKU** ★ | 일반 |
| Product + JSON variants | MVP |
| ProductGroup + 독립 Products | 다른 상품으로 분리 |
| EAV (entity-attribute-value) | enterprise — 옵션 무한 |

### 2.4 트레이드오프

- SKU 분리 = 정확 + 카르테시안 곱 (3 size × 5 color × 3 fit = 45 SKU).
- JSON = 단순 + 검색 / 재고 X.

---

## 3. 데이터 구조

```sql
products (
    id, name, status, base_price_krw, ...
);

product_options (
    id, product_id, name, sort_order
);

product_option_values (
    id, option_id, value, sort_order
);

product_skus (
    id, product_id, sku_code, price_krw, ...
);

product_sku_options (
    sku_id, option_value_id      -- M:N
);

inventory (
    sku_id, on_hand, ...
);
```

자세히: [[../database/products-table]] · [[../database/product-options-table]] · [[../database/inventory-table]].

---

## 4. 함정

### 함정 1 — Product 단위 재고
red XL SOLD_OUT 가 blue M 도 표시.
→ SKU 단위 inventory.

### 함정 2 — 옵션 hardcode
새 옵션 추가 시 schema 변경.
→ EAV 또는 product_options.

### 함정 3 — SKU 폭증 (3×5×4×3 = 180)
한 product 에 180 SKU.
→ 필수 옵션만 + lazy generation.

### 함정 4 — 옵션 값 i18n X
글로벌 진입 시 옵션 이름 (size) 번역 X.
→ option_value_i18n 테이블.

---

## 5. 다른 컨텍스트

### 5.1 무신사 / 신발
size + width × color → SKU 폭증 → bitmap 으로 검색.

### 5.2 음원 (멜론)
SKU = 음원 1개 (옵션 없음).

### 5.3 강의 (인프런)
SKU = 강의 1개 (옵션 없음).

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[../database/products-table]]
- [[../database/product-options-table]]
- [[inventory-strategy]]
