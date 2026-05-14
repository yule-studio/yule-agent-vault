---
title: "refunds 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - refund
---

# refunds 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 1 payment = N refunds (부분 환불 여러번).

---

## 1. Schema

```sql
-- V34__create_refunds.sql
CREATE TABLE refunds (
    id              CHAR(26) PRIMARY KEY,
    payment_id      CHAR(26) NOT NULL REFERENCES payments(id),
    amount          NUMERIC(15, 4) NOT NULL CHECK (amount > 0),
    currency        CHAR(3) NOT NULL,
    reason          VARCHAR(500) NOT NULL,
    initiated_by    CHAR(26) NOT NULL,            -- user / admin id
    initiated_role  VARCHAR(20) NOT NULL,         -- USER / ADMIN / SYSTEM
    pg_cancel_key   VARCHAR(100),                  -- PG 환불 키
    pg_response     JSONB,
    idempotency_key VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,

    CONSTRAINT chk_refund_status CHECK
        (status IN ('PENDING', 'COMPLETED', 'FAILED'))
);

CREATE INDEX ix_refunds_payment ON refunds (payment_id, occurred_at DESC);
CREATE UNIQUE INDEX ux_refunds_idempotency
    ON refunds (idempotency_key) WHERE idempotency_key IS NOT NULL;
```

---

## 2. 컬럼 "왜"

### 2.1 1:N (vs payment 의 refund_total)

- 부분 환불 여러번 (5만원 → 1만원 1회 → 2만원 1회).
- 각 refund 의 사유 / 시점 audit.

### 2.2 `initiated_by` + `initiated_role`

- 사용자 자체 환불 / admin 처리 / 자동 (PG webhook cancel) 구분.
- 5년 audit.

### 2.3 `pg_cancel_key`

- PG 의 cancel 응답 key — 다시 조회 시 사용.

### 2.4 `idempotency_key`

- 중복 환불 방어.

---

## 3. 함정

### 함정 1 — payment.amount 변경 시도 (refunded)
환불 시 payment 의 amount 를 차감 → 회계 mismatch.
→ payment.amount 는 immutable, refunds 합산으로 잔여 계산.

### 함정 2 — 환불 합계 > payment.amount
부정합.
→ application layer 검증 + CHECK.

### 함정 3 — reason 누락
audit 추적 X.
→ NOT NULL.

### 함정 4 — admin 만 자동 (사용자 self-refund X)
사용자가 7일 단순변심 자동 못 함.
→ initiated_role = USER 허용 (조건 부).

---

## 4. 관련

- [[database|↑ hub]]
- [[payments-table]]
- [[../design-decisions/refund-policy]]
- [[../implementation/refund-impl]]
