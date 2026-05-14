---
title: "정산 정책 — PG D+3 → 가맹점 D+5"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - settlement
---

# 정산 정책 — PG D+3 → 가맹점 D+5

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> PG → 가맹점 의 정산 주기 + 대사 (서버 vs PG) + 환불 영향.

---

## 1. 본 vault 결정

```
PG 정산 흐름:
T (결제일)
→ T+1 (카드사 → PG)
→ T+3 (PG → 가맹점 통장 입금)
→ T+5 (회사 회계 정산)
```

- daily 대사 (서버 payment 합계 = PG 정산 금액).
- 환불 시 settlement_adjustment row INSERT (음수 amount).
- 월 마감 — 운영 dashboard.

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 정산 row 별도

- payment row 만 보면 매출 / fee / 환불 분리 X.
- settlement = "T일 매출 X 원, fee Y 원, 환불 Z 원, 입금 W 원".

### 2.2 안 하면

| 잘못 | 사고 |
| --- | --- |
| 매출 = SUM(payment.amount) 가정 | 환불 빼야 함 |
| PG fee 미반영 | 회계 부풀려짐 |
| 대사 X | PG 가 보낸 정산금 vs 서버 매출 mismatch — 일치 안 함 |
| 환불 시 settlement 안 조정 | 다음 달 정산 mismatch |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **per-payment settlement row** ★ | 일반 |
| daily aggregate | scale ↑ |
| 자체 정산 시스템 | 큰 platform (multi-vendor) |

---

## 3. 데이터

```sql
CREATE TABLE settlements (
    id           CHAR(26) PRIMARY KEY,
    payment_id   CHAR(26) NOT NULL UNIQUE,
    amount_gross BIGINT NOT NULL,         -- 결제 총액
    pg_fee       BIGINT NOT NULL,         -- PG 수수료
    amount_net   BIGINT NOT NULL,         -- 입금액
    settled_at   TIMESTAMPTZ,              -- PG 입금 일 (확정)
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE settlement_adjustments (
    id           CHAR(26) PRIMARY KEY,
    settlement_id CHAR(26) NOT NULL REFERENCES settlements(id),
    type         VARCHAR(20) NOT NULL,    -- REFUND / CHARGEBACK / FEE_ADJUST
    amount       BIGINT NOT NULL,          -- 음수 가능
    reason       VARCHAR(500),
    occurred_at  TIMESTAMPTZ NOT NULL
);
```

---

## 4. 대사 (Reconciliation)

```
매일 00:00 cron:
1. PG 의 정산 API → 어제 입금 목록 가져옴
2. 서버 settlements 의 어제 row 합계
3. 차이 > 0 → Slack alert
4. 매주 admin 확인
```

자세히: [[../operations/reconciliation]].

---

## 5. 함정

### 함정 1 — fee 미저장
PG fee 별도 row 없음 → 회계 추적 X.
→ pg_fee 컬럼 + audit.

### 함정 2 — 환불 시 settlement 안 조정
다음 달 정산 mismatch.
→ settlement_adjustments INSERT.

### 함정 3 — 대사 자동화 X
mismatch 발견 늦음.
→ daily cron + alert.

### 함정 4 — PG 정산 webhook 신뢰 X
PG 가 보낸 정산 vs 입금 실제 → 차이 가능.
→ 통장 입금 API 와 이중 대사.

---

## 6. 다른 컨텍스트

### 6.1 multi-vendor (마켓플레이스)
vendor 별 정산 row + 사이트 fee + vendor 입금 자동.

### 6.2 글로벌 (Stripe Connect)
multi-currency 정산 + 환율 lock 시점.

### 6.3 B2B (NET 30)
invoice 단위 정산 — 후불 30일.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[pg-selection#6]]
- [[refund-policy]]
- [[../operations/reconciliation]]
- [[../implementation/settlement-impl]]
