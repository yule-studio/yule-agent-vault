---
title: "정산 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:42:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - settlement
---

# 정산 구현

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. PaymentApproved → settlement row

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentApproved(PaymentApproved ev) {
    var pgFee = calculatePgFee(ev.amount(), ev.pgProvider());
    var net = ev.amount().subtract(pgFee);

    settlements.save(new Settlement(
        ids.next(), ev.paymentId(),
        ev.amount(), pgFee, net,
        null,                       // settled_at — PG 입금 후 update
        clock.now()));
}
```

---

## 2. PaymentCanceled → adjustment

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentCanceled(PaymentCanceled ev) {
    settlements.addAdjustment(
        ev.paymentId(),
        "REFUND",
        ev.refundAmount().negate(),
        ev.reason(),
        clock.now());
}
```

---

## 3. Daily 대사 cron

```java
@Scheduled(cron = "0 0 1 * * *")    // 매일 01:00
public void dailyReconciliation() {
    var yesterday = LocalDate.now().minusDays(1);
    var serverTotal = settlements.sumNetAmountByDate(yesterday);
    var pgTotal = tossApi.getSettledAmount(yesterday);

    if (!serverTotal.equals(pgTotal)) {
        slack.alert("settlement mismatch", Map.of(
            "date", yesterday,
            "serverTotal", serverTotal,
            "pgTotal", pgTotal,
            "diff", pgTotal.subtract(serverTotal)));
    }
}
```

자세히: [[../operations/reconciliation]].

---

## 4. 함정

### 함정 1 — fee 미저장
회계 추적 X.
→ pg_fee 컬럼.

### 함정 2 — 환불 adjust 누락
mismatch.
→ adjustment row.

### 함정 3 — 대사 자동화 X
발견 늦음.
→ daily cron + alert.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/settlement-policy]]
- [[../operations/reconciliation]]
