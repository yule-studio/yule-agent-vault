---
title: "PII 암호화 — email / phone / token"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:32:00+09:00
tags: [backend, java-spring, api-design, notification, security, pii]
---

# PII 암호화 — email / phone / token

**[[security|↑ hub]]**

---

| 데이터 | 처리 |
| --- | --- |
| 사용자 email (template 변수) | signup recipe 의 암호화 따름 |
| FCM/APNs token | KMS encrypt (option) |
| variables JSONB 안 PII | type 별 정책 (예: 카드 마지막 4자리는 OK, 풀카드 X) |
| notifications 의 body | 평문 OK but 사용자별 권한 |
| audit log 의 IP | 5년 보유 |

---

## 1. variables 안 PII 정책

```java
public Map<String, Object> sanitize(NotificationType type, Map<String, Object> vars) {
    // 민감 변수 mask
    if (type == PAYMENT_APPROVED) {
        if (vars.containsKey("cardNumber")) {
            vars.put("cardNumber", "****" + lastFour((String) vars.get("cardNumber")));
        }
    }
    return vars;
}
```

→ outbox INSERT 전 sanitize.

---

## 2. 함정

- raw card / password / SSN 을 variables 에 넣음 → DB dump 유출.
- email 평문 — 단순 OK but 다중 사용자 query 시 노출.
- audit log 5년 — anonymize cron 옵션.

---

## 관련

- [[security|↑ hub]]
- [[../../signup/security/pii-encryption|↗ signup PII]]
- [[../../product/security/pii-encryption|↗ product PII]]
