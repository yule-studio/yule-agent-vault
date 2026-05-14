---
title: "Presence 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:38:00+09:00
tags: [backend, java-spring, api-design, chat, pitfalls, presence]
---

# Presence 함정

**[[pitfalls|↑ hub]]**

---

1. **DB 에 presence write** → 부하 폭주 (분당 수만 write).
2. **Redis TTL 없음** → 영구 ONLINE 표시.
3. **heartbeat 1초 마다** → 부하 ↑.
4. **typing broadcast 매번** → debounce 1s.
5. **friend 1만+ 명에게 broadcast** → 개별 send 시 latency.
   → batch / fanout 분리.
6. **"마지막 접속" privacy 무시** → 사용자 preference 무시.
7. **subscriber 안 함** → presence change 가 다른 노드 못 받음.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/presence-strategy]]
- [[../implementation/presence-impl]]
