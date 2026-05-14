---
title: "Connection / WebSocket 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:32:00+09:00
tags: [backend, java-spring, api-design, chat, pitfalls]
---

# Connection / WebSocket 함정

**[[pitfalls|↑ hub]]**

---

1. **HTTPS/WSS 없이 production** → MITM.
2. **proxy idle timeout** (nginx 60s) → 자동 disconnect → heartbeat 필요.
3. **Sticky session 없음** → 매번 다른 노드 → SimpUserRegistry 깨짐.
4. **`setAllowedOrigins("*")`** → CSRF.
5. **CONNECT 후 JWT 검증 안 함** → anonymous user.
6. **SUBSCRIBE 시 room member 검증 X** → privacy 누설.
7. **DISCONNECT 시 cleanup 안 함** → Redis presence / session 누적.
8. **OS file descriptor 1024 기본** → 1만 connection 못 받음.
9. **Reconnect 시 새 SessionId** → 옛 session cleanup 안 함.
10. **WebSocket 안 모니터링** → 사고 모름.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../security/websocket-auth]]
- [[../operations/scaling]]
