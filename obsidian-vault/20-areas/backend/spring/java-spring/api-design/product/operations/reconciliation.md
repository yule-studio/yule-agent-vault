---
title: "PG 대사 (Reconciliation) — daily"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:08:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - operations
  - reconciliation
---

# PG 대사 (Reconciliation) — daily

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[operations|↑ hub]]**

---

## 1. 무엇

"서버의 `payments + adjustments` 합계 = PG 가 정산한 금액"  
일치하지 않으면 매출 / 회계 mismatch → 사고 (cron / batch 누락 / webhook 손실).

---

## 2. Cron

```java
@Scheduled(cron = "0 0 1 * * *")     // 매일 01:00
public void reconcile() {
    var yesterday = LocalDate.now(ZoneId.of("Asia/Seoul")).minusDays(1);

    // 서버 계산
    var serverTotal = settlements.sumNetByDate(yesterday);
    var serverPayCount = payments.countDoneByDate(yesterday);

    // PG 조회
    var pgReport = tossApi.getSettlementReport(yesterday);

    var diff = pgReport.netAmount().subtract(serverTotal);
    var paymentDiff = pgReport.paymentCount() - serverPayCount;

    metrics.gauge("product_reconciliation_diff_krw", diff.amountMinorUnit());

    if (diff.amountMinorUnit() != 0 || paymentDiff != 0) {
        slack.alert("settlement mismatch", Map.of(
            "date", yesterday,
            "serverTotal", serverTotal,
            "pgTotal", pgReport.netAmount(),
            "diff", diff,
            "serverCount", serverPayCount,
            "pgCount", pgReport.paymentCount()));
    }
}
```

---

## 3. mismatch 패턴

| 원인 | 검출 |
| --- | --- |
| webhook 누락 (서버 모름) | PG count > 서버 count |
| 환불 settlement adjust 누락 | 서버 net > PG net |
| PG fee 계산 오류 | 차이가 fee 만큼 |
| 시간대 (KST vs UTC) | 자정 근처 결제 차이 |
| 환불 안 됨 (서버 보냈는데 PG X) | 환불 mismatch |

---

## 4. 수동 대사

```sql
-- 어제 결제 합계
SELECT
    SUM(amount) FILTER (WHERE status='DONE') AS gross,
    SUM(pg_fee) AS fee,
    COUNT(*) FILTER (WHERE status='DONE') AS cnt
FROM payments p
JOIN settlements s ON s.payment_id = p.id
WHERE p.approved_at >= '2026-05-13 00:00:00+09'
  AND p.approved_at <  '2026-05-14 00:00:00+09';

-- 어제 환불
SELECT SUM(amount) FROM refunds
WHERE occurred_at >= '2026-05-13 00:00:00+09'
  AND occurred_at <  '2026-05-14 00:00:00+09';
```

---

## 5. 함정

### 함정 1 — TZ 혼동 (KST vs UTC)
자정 근처 결제 분류 오류.
→ KST 명시.

### 함정 2 — 환불 adjust 시점
환불 webhook 도착이 정산 cron 후.
→ 다음 날 batch 에 포함.

### 함정 3 — PG api 한도
대용량 매출 시 PG report API hourly 한도.
→ daily batch + cache.

---

## 6. 관련

- [[operations|↑ hub]]
- [[observability]]
- [[runbook]]
- [[../design-decisions/settlement-policy]]
- [[../implementation/settlement-impl]]
