---
title: "notification implementation hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:40:00+09:00
tags: [backend, java-spring, api-design, notification, implementation]
---

# notification implementation hub

**[[../notification|↑ hub]]**

---

| 노트 | 영역 |
| --- | --- |
| [[outbox-worker-impl]] ★ | Worker (SKIP LOCKED + exp backoff) |
| [[channel-router-impl]] ★ | Router |
| [[fcm-channel-impl]] ★ | Firebase Admin SDK |
| [[apns-channel-impl]] ★ | APNs HTTP/2 + JWT |
| [[webpush-channel-impl]] | VAPID |
| [[ses-email-channel-impl]] ★ | AWS SES |
| [[slack-channel-impl]] | admin webhook |
| [[in-app-channel-impl]] | DB row |
| [[device-management-impl]] ★ | token CRUD |
| [[preference-impl]] | settings |
| [[template-impl]] | i18n + 변수 |
| [[kafka-integration]] | F8+ |

---

## 관련

- [[../notification|↑ hub]]
- [[../architecture]]
