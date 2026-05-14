---
title: "notification 도메인 이벤트"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - event
---

# notification 도메인 이벤트

**[[domain-model|↑ hub]]**

---

## 1. 이벤트

| Event | 트리거 | 처리 |
| --- | --- | --- |
| NotificationCreated | outbox INSERT | audit |
| NotificationSent | 채널 ack | metric + audit |
| NotificationFailed | transient retry max 초과 | DLQ + alert |
| NotificationDlq | DLQ row 생성 | admin alert |
| DeviceRegistered | 사용자 첫 등록 | audit |
| DeviceInvalidated | UNREGISTERED 응답 | audit + cleanup |
| DeviceTokenRotated | FCM rotate | audit |

---

## 2. 본 모듈 외부 (다른 도메인) 의 trigger

```
도메인 event (예: OrderPaid) → 도메인 listener → outbox INSERT → NotificationCreated
```

→ notification 모듈 의 listener 는 외부 도메인 이벤트 받음.

---

## 3. 관련

- [[domain-model|↑ hub]]
- [[../implementation/outbox-worker-impl]]
- [[../security/audit-logging]]
