---
title: "notification domain-model hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:02:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
---

# notification domain-model hub

**[[../notification|↑ hub]]**

---

## 1. Aggregate

| Aggregate | 책임 | 노트 |
| --- | --- | --- |
| **Notification** (outbox row) | 발송 lifecycle | [[notification-aggregate]] |
| **UserDevice** | 사용자 별 device | [[user-device-aggregate]] |

---

## 2. VOs / Events / Ports

- [[value-objects]] — NotificationId / TemplateKey / DeviceToken
- [[domain-events]] — NotificationCreated / Sent / Failed / DLQ
- [[repository-ports]] — Repository + NotificationChannel port
- [[aggregate-boundaries]]

---

## 3. 관련

- [[../notification|↑ hub]]
- [[../database/database]]
- [[../architecture]]
