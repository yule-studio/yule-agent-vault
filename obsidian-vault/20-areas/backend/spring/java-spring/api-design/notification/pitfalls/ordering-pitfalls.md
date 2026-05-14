---
title: "Ordering 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:30:00+09:00
tags: [backend, java-spring, api-design, notification, pitfalls, ordering]
---

# Ordering 함정

**[[pitfalls|↑ hub]]**

---

1. **Concurrent worker** 같은 사용자의 알림 2개 동시 → 순서 X.
2. **retry 시 순서 깨짐** A retry 후 B 보다 늦게 도착.
3. **chat 메시지 순서** "오늘 만나자" 가 "안녕" 보다 먼저.
4. **payment APPROVED / CANCELED 순서** 반대 도착 시 사용자 confusion.
5. **partition key 잘못** (random) → 순서 보장 X.
6. **multi-channel 의 다른 latency** push 가 email 보다 먼저 — OK (의미적 같은 alert).

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/ordering]]
