---
title: "Token 함정 (FCM/APNs)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:28:00+09:00
tags: [backend, java-spring, api-design, notification, pitfalls, token]
---

# Token 함정 (FCM/APNs)

**[[pitfalls|↑ hub]]**

---

1. **UNREGISTERED 무시** → 영구 무효 token 발송 → 비용.
2. **token UNIQUE 없음** → 같은 device 의 multi row.
3. **token 평문 저장** → DB dump 시 spam 발송.
4. **rotation 안 함** → 옛 token 무한 보관.
5. **device 양도 처리 X** → 옛 user 에게 알림.
6. **logout 시 deactivate 안 함** → 다른 user 가 device 받으면 옛 user 의 알림 봄.
7. **90일 cleanup 없음** → DB 누적.
8. **locale / timezone 없음** → i18n + quiet hours X.
9. **device_id (FE UUID) 없음** → 같은 device 의 token rotation 인지 구분 X.
10. **MulticastMessage 안 씀** (1 사용자 multi device) → API 호출 N배.
11. **FCM auth error (service account)** → admin alert 없으면 모든 발송 fail.
12. **APNs sandbox / production 혼동** → BadDeviceToken.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/device-management]]
- [[../security/device-token-rotation]]
- [[../implementation/fcm-channel-impl]]
- [[../implementation/apns-channel-impl]]
