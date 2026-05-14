---
title: "Audit Logging — admin / system action 5년"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - audit
---

# Audit Logging — admin / system action 5년

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

---

## 1. 무엇을 audit

| 영역 | 보유 |
| --- | --- |
| 상품 등록 / 가격 변경 / 단종 | 5년 |
| admin 환불 / 모더 / 상태 변경 | 5년 |
| 결제 시도 (성공 / 실패) | 5년 (전자상거래법) |
| webhook 수신 | 30일 (별도 webhook_events) |
| 다운로드 (디지털) | 3년 (분쟁 윈도우) |
| 로그인 / 인증 | signup 의 audit ([[../../signup/security/audit-logging|↗]]) |

---

## 2. 테이블

```sql
CREATE TABLE product_audit_log (
    id              CHAR(26) PRIMARY KEY,
    event_type      VARCHAR(50) NOT NULL,
    actor_id        CHAR(26),
    actor_role      VARCHAR(20),       -- USER / ADMIN / SYSTEM
    target_id       CHAR(26),
    target_type     VARCHAR(30),       -- PRODUCT / ORDER / PAYMENT / REFUND
    ip_address      VARCHAR(45),
    user_agent      VARCHAR(500),
    metadata        JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_audit_event_at ON product_audit_log (event_type, occurred_at DESC);
CREATE INDEX ix_audit_actor ON product_audit_log (actor_id, occurred_at DESC);
CREATE INDEX ix_audit_target ON product_audit_log (target_type, target_id, occurred_at DESC);
```

---

## 3. Logger

```java
@Component
@RequiredArgsConstructor
class ProductAuditLogger {

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onProductPriceChanged(ProductPriceChanged ev) {
        save("PRODUCT_PRICE_CHANGED", ev.changedBy(), "ADMIN",
             ev.productId(), "PRODUCT",
             Map.of("oldPrice", ev.oldPrice(), "newPrice", ev.newPrice()));
    }

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onAdminRefund(AdminRefundIssued ev) {
        save("ADMIN_REFUND", ev.adminId(), "ADMIN",
             ev.paymentId(), "PAYMENT",
             Map.of("amount", ev.amount(), "reason", ev.reason()));
    }

    private void save(...) { ... }
}
```

---

## 4. 알람

| Event | 알람 |
| --- | --- |
| ADMIN_REFUND amount > 100만원 | Slack #payment-alerts |
| ADMIN_DELETE PRODUCT | Slack #admin-actions |
| PAYMENT_FAILED spike (10 분 50+) | Slack #payment-alerts |
| AUTO_DIGITAL_REVOKE spike | Slack #fraud |

---

## 5. Cleanup (보유 기간)

```sql
-- 5년 지난 audit
DELETE FROM product_audit_log
WHERE occurred_at < now() - INTERVAL '5 years';
```

→ cold storage (S3) 로 archive 후 삭제 옵션.

---

## 6. 함정

### 함정 1 — 트랜잭션 안 audit
rollback 시 audit 사라짐.
→ AFTER_COMMIT 또는 REQUIRES_NEW.

### 함정 2 — PII 직접 저장
사용자 주소 / 카드번호 audit 에.
→ ID + metadata key (값 X).

### 함정 3 — 보유 기간 X
DB 무한 누적.
→ cleanup cron + cold storage.

### 함정 4 — actor_id NULL (SYSTEM action)
"누가" 모름.
→ actor_role='SYSTEM' + 명시.

---

## 7. 관련

- [[security|↑ hub]]
- [[../../signup/security/audit-logging|↗ signup audit]]
- [[../database/payment-transactions-table]]
- [[../design-decisions/refund-policy]]
