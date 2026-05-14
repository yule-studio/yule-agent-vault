---
title: "chat testing hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:10:00+09:00
tags: [backend, java-spring, api-design, chat, testing]
---

# chat testing hub

**[[../chat|↑ hub]]**

| 노트 | 영역 |
| --- | --- |
| [[test-scenarios]] | AC 매핑 |
| [[unit-tests]] | 도메인 |
| [[integration-tests]] | Testcontainers + WebSocketStompClient |
| [[load-tests]] | k6 동시 1만 connection |
| [[e2e-tests]] | 멀티 user / 멀티 device |

---

## 도구

- JUnit 5 / Mockito / AssertJ
- Spring `WebSocketStompClient` (test)
- Testcontainers (PostgreSQL + Redis + RabbitMQ)
- Awaitility (비동기 broadcast)
- k6 / Gatling (load)
- Playwright (e2e — browser)

---

## 관련

- [[../chat|↑ hub]]
- [[../requirements]]
