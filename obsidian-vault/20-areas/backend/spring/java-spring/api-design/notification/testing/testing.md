---
title: "notification testing hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:06:00+09:00
tags: [backend, java-spring, api-design, notification, testing]
---

# notification testing hub

**[[../notification|↑ hub]]**

---

| 노트 | 영역 |
| --- | --- |
| [[test-scenarios]] | AC 매핑 |
| [[unit-tests]] | aggregate / VOs |
| [[integration-tests]] | Testcontainers + WireMock |
| [[channel-mock-tests]] | FCM/APNs/SES mock |

---

## 도구

- JUnit 5 + Mockito + AssertJ
- Testcontainers (PostgreSQL + Redis)
- WireMock (FCM HTTP + SES)
- LocalStack (SES)
- Pushy MockApnsServer (APNs)
- Awaitility (worker / async)

---

## 관련

- [[../notification|↑ hub]]
- [[../requirements]]
