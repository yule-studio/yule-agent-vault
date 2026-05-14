---
title: "coupons + coupon_redemptions 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:03:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - coupon
---

# coupons + coupon_redemptions 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 쿠폰 정의 + 사용자 발급 + 사용 추적.

---

## 1. Schema

```sql
-- V38__create_coupons.sql
CREATE TABLE coupons (
    id              CHAR(26) PRIMARY KEY,
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    discount_type   VARCHAR(20) NOT NULL,            -- FIXED_AMOUNT / PERCENTAGE / FREE_SHIPPING
    discount_value  NUMERIC(15, 4) NOT NULL,         -- 5000 (FIXED) or 0.10 (PCT)
    max_discount    NUMERIC(15, 4),                   -- PCT 의 max cap (5000원)
    min_order       NUMERIC(15, 4) NOT NULL DEFAULT 0,
    usage_limit     INTEGER,                          -- 전체 발급 가능 수 (NULL = 무제한)
    usage_count     INTEGER NOT NULL DEFAULT 0,
    per_user_limit  INTEGER NOT NULL DEFAULT 1,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    starts_at       TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_coupons_active ON coupons (expires_at) WHERE active;

CREATE TABLE coupon_redemptions (
    id              CHAR(26) PRIMARY KEY,
    coupon_id       CHAR(26) NOT NULL REFERENCES coupons(id),
    user_id         CHAR(26) NOT NULL,
    order_id        CHAR(26) NOT NULL REFERENCES orders(id),
    discount_amount NUMERIC(15, 4) NOT NULL,
    used_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    restored_at     TIMESTAMPTZ,                      -- 환불 시 복원 시각

    UNIQUE (user_id, coupon_id, order_id)
);

CREATE INDEX ix_redemptions_user ON coupon_redemptions (user_id, used_at DESC);
```

---

## 2. 컬럼 "왜"

### 2.1 `discount_type` + `max_discount`

- PCT 의 cap (10% but max 5000원).
- 큰 금액 결제 시 무한 할인 방지.

### 2.2 `usage_count` (vs DB UPDATE WHERE remaining > 0)

- atomic 차감 — race 방지.
- usage_limit 와 비교.

### 2.3 `per_user_limit`

- 1명이 같은 쿠폰 N번 사용 — 정책.
- coupon_redemptions UNIQUE 로 enforce.

### 2.4 `restored_at` (환불 복원)

- 환불 시 쿠폰 복원 정책 (선택) — restored_at = now() 면 다시 사용 가능.
- 정책: 복원 X (기본) vs 복원 (이벤트 쿠폰).

---

## 3. 함정

### 함정 1 — usage_count 동시 차감 X
race → usage_limit 초과.
→ UPDATE WHERE usage_count < usage_limit + version.

### 함정 2 — per_user_limit X
1명이 무한 사용.
→ UNIQUE (user_id, coupon_id, order_id) + count.

### 함정 3 — 만료 검증 X
만료된 쿠폰 사용.
→ expires_at > now() 검증.

### 함정 4 — PCT max cap X
1억 결제의 10% = 1천만원 할인.
→ max_discount.

### 함정 5 — 환불 복원 정책 명확 X
사용자가 쿠폰 무한 재사용.
→ 정책 (기본 복원 X).

---

## 4. 관련

- [[database|↑ hub]]
- [[orders-table]]
- [[../design-decisions/pricing-strategy]]
- [[../implementation/coupon-impl]]
