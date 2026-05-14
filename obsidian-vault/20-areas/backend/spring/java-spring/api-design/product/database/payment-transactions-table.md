---
title: "payment_transactions 테이블 — 시도별 audit"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:53:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - payment
  - audit
---

# payment_transactions 테이블 — 시도별 audit

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 결제 시도 1번 = 1 row. 실패 / 재시도 / cancel 까지 모든 trace.

---

## 1. Schema

```sql
-- V33__create_payment_transactions.sql
CREATE TABLE payment_transactions (
    id              CHAR(26) PRIMARY KEY,
    payment_id      CHAR(26) NOT NULL REFERENCES payments(id),
    attempt_type    VARCHAR(30) NOT NULL,             -- CONFIRM / CANCEL / WEBHOOK
    pg_provider     VARCHAR(20) NOT NULL,
    pg_response_code VARCHAR(20),
    pg_response_msg VARCHAR(500),
    pg_raw_request  JSONB,
    pg_raw_response JSONB,
    success         BOOLEAN NOT NULL,
    duration_ms     INTEGER,
    attempted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        CHAR(26),                          -- user / admin (수동 cancel)
    metadata        JSONB
);

CREATE INDEX ix_pt_payment_attempted ON payment_transactions (payment_id, attempted_at DESC);
CREATE INDEX ix_pt_failed ON payment_transactions (attempted_at) WHERE success = FALSE;
```

---

## 2. 컬럼 "왜"

### 2.1 왜 별도 테이블 (payments 의 status 만 아님)

- payment 의 status 는 현재 상태 — 시도 1번 / 5번 차이 X.
- payment_transactions 는 모든 시도 history.

### 2.2 `pg_raw_request / response`

- 분쟁 시 PG 가 무엇을 받고 무엇을 보냈는지.
- raw = JSONB (모든 PG 호환).

### 2.3 `duration_ms`

- PG 응답 시간 통계 (p95 / p99).
- 운영 SLO 모니터링.

### 2.4 partial index `failed`

- 실패 빈도 알람 빠르게.

---

## 3. 사용 예

```
1. 사용자 /confirm
2. payment_transactions INSERT (attempt_type=CONFIRM, pg_request=..., pg_response=..., success=true, duration=1234ms)
3. 환불 시 INSERT (attempt_type=CANCEL, ...)
4. webhook 도 INSERT (attempt_type=WEBHOOK, ...)
```

---

## 4. 함정

### 함정 1 — 실패 raw 안 저장
사고 시 PG 가 무엇을 응답했는지 모름.
→ pg_raw_response.

### 함정 2 — duration 누락
PG SLO 추적 X.
→ duration_ms.

### 함정 3 — actor_id 누락
admin 수동 cancel 시 누가 했는지 모름.
→ actor_id.

### 함정 4 — payments 의 status 만 의존
시도 / 재시도 history X.
→ 별도 transactions.

---

## 5. 관련

- [[database|↑ hub]]
- [[payments-table]]
- [[webhook-events-table]]
- [[../security/audit-logging]]
