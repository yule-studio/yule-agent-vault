---
title: "Scale 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:40:00+09:00
tags: [backend, java-spring, api-design, chat, pitfalls, scale]
---

# Scale 함정

**[[pitfalls|↑ hub]]**

---

1. **단일 노드 SimpleBroker production** → 1만 connection 초과 시 죽음.
2. **sticky session 없이 다중 노드** → session 끊김.
3. **Redis 1 노드** → SPOF.
4. **RabbitMQ 단일 instance** → cluster 필수.
5. **graceful disconnect 없음** → rolling deploy 시 사용자 끊김.
6. **partition 안 함** → messages 테이블 100억 row.
7. **JVM heap 작게** (1GB) → GC pause spike.
8. **monitoring 없음** → scale 시점 모름.
9. **Tomcat thread pool 작음** → blocking I/O 시 한도.
10. **WebFlux / Netty 검토 안 함** → 5만 connection 이상 어려움.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/scale-strategy]]
- [[../operations/scaling]]
