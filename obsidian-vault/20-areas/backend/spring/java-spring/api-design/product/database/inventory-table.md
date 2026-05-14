---
title: "inventory 테이블 — SKU 단위 재고"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:42:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - inventory
---

# inventory 테이블 — SKU 단위 재고

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> SKU 단위 재고 + Redis primary + DB source of truth + version optimistic lock.

---

## 1. Schema

```sql
-- V25__create_inventory.sql
CREATE TABLE inventory (
    sku_id       CHAR(26) PRIMARY KEY REFERENCES product_skus(id),
    on_hand      INTEGER NOT NULL CHECK (on_hand >= 0),
    reserved     INTEGER NOT NULL DEFAULT 0 CHECK (reserved >= 0),
    low_threshold INTEGER NOT NULL DEFAULT 10,
    version      BIGINT NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_inv_reserved CHECK (reserved <= on_hand)
);

CREATE INDEX ix_inv_low ON inventory (sku_id) WHERE on_hand <= low_threshold;
```

---

## 2. 컬럼 "왜"

### 2.1 `on_hand` vs `reserved`

- `on_hand` = 실제 창고 수량.
- `reserved` = 주문 PENDING 으로 임시 차감 (결제 timeout 시 복원).
- `available = on_hand - reserved`.

### 2.2 `low_threshold` + partial index

- 재고 부족 알림 (admin 에게).
- partial index = 대부분 row 는 skip, 부족한 것만 빠르게 조회.

### 2.3 `version BIGINT`

- Optimistic lock — Redis 와 DB 동기 race 방지.

---

## 3. Redis ↔ DB 동기

```
주문 차감:
1. Redis Lua DECRBY (atomic)
2. order INSERT (status=PENDING)
3. AFTER_COMMIT: DB inventory UPDATE SET reserved += qty

결제 승인:
4. AFTER_COMMIT: DB inventory UPDATE SET on_hand -= qty, reserved -= qty

결제 실패 / 타임아웃:
5. order CANCEL: Redis INCRBY (복원)
6. DB inventory UPDATE SET reserved -= qty

매 15분 cron:
7. DB on_hand - reserved 와 Redis 값 비교 → mismatch 시 alert
```

자세히: [[../design-decisions/inventory-strategy]].

---

## 4. 함정

### 함정 1 — Redis 만 (DB X)
Redis 장애 시 재고 손실.
→ DB 동기.

### 함정 2 — reserved 누락 (on_hand 만)
주문 PENDING 시점 차감 → 결제 실패 시 복원 어려움.
→ on_hand vs reserved 분리.

### 함정 3 — version 없음
Optimistic lock X — race 시 lost update.
→ version + WHERE version=?.

### 함정 4 — 음수 (CHECK 누락)
on_hand = -1 → 표시 오류.
→ CHECK 제약.

---

## 5. 관련

- [[database|↑ hub]]
- [[products-table#sku]]
- [[../design-decisions/inventory-strategy]]
- [[../../distributed-lock|↗ distributed-lock]]
