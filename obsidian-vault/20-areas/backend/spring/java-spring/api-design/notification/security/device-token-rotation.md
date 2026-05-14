---
title: "Device token 회전 — FCM/APNs 만료 처리"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:30:00+09:00
tags: [backend, java-spring, api-design, notification, security, token]
---

# Device token 회전 — FCM/APNs 만료 처리 ★

**[[security|↑ hub]]**

---

## 1. token 만료 / 회전 원인

| 원인 | FCM | APNs |
| --- | --- | --- |
| App 재설치 | 새 token | 새 token |
| App 캐시 삭제 | 새 token | 새 token (가끔) |
| OS 업그레이드 | 가끔 | 가끔 |
| FCM 자체 rotate (보안 정책) | 발생 | — |
| 사용자가 알림 권한 거부 | 무효 | 무효 |

→ FE 가 매 시작 시 token check + 변경 시 PATCH.

---

## 2. 응답별 처리

### FCM

| Response | 처리 |
| --- | --- |
| `UNREGISTERED` | 즉시 deactivate (`invalid_reason='UNREGISTERED'`) |
| `INVALID_ARGUMENT` | deactivate (잘못된 token) |
| `QUOTA_EXCEEDED` | 일시 — retry |
| `INTERNAL` | 일시 — retry |
| `THIRD_PARTY_AUTH_ERROR` | service account 문제 — admin alert |

### APNs

| Status | 처리 |
| --- | --- |
| 410 (BadDeviceToken) | 즉시 deactivate |
| 410 (Unregistered) | 즉시 deactivate |
| 429 (TooManyRequests) | 일시 — retry |
| 500 (InternalServerError) | 일시 — retry |
| 403 (BadCertificate) | service config 문제 — admin alert |

---

## 3. 코드

```java
catch (FirebaseMessagingException e) {
    if (e.getMessagingErrorCode() == MessagingErrorCode.UNREGISTERED ||
        e.getMessagingErrorCode() == MessagingErrorCode.INVALID_ARGUMENT) {
        deviceRepo.markInvalid(device.id(), e.getMessagingErrorCode().name());
        return DeliveryResult.permanent(...);
    }
    return DeliveryResult.transient(...);
}
```

---

## 4. token 보관 보안

```sql
-- option A: 단순 (평문 + DB 접근 제어)
token TEXT NOT NULL

-- option B: pgcrypto (KMS key)
token_encrypted BYTEA
```

→ 본 vault: production 은 KMS encryption 권장 (PII 와 비슷한 민감도).

---

## 5. 함정

- UNREGISTERED 무시 — 영구 무효 token 에 발송 → 비용.
- token 평문 + git log — secrets 유출.
- rotate 시 옛 token 무한 보관 → DB 누적.

---

## 6. 관련

- [[security|↑ hub]]
- [[../design-decisions/device-management]]
- [[../implementation/fcm-channel-impl]]
- [[../implementation/apns-channel-impl]]
- [[../database/user-devices-table]]
