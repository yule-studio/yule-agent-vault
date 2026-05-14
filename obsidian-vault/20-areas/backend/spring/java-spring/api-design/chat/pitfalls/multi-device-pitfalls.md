---
title: "Multi-device 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:36:00+09:00
tags: [backend, java-spring, api-design, chat, pitfalls, multi-device]
---

# Multi-device 함정

**[[pitfalls|↑ hub]]**

---

1. **본인 메시지 echo 안 함** → 폰에서 send 후 폰 화면에 안 표시.
2. **read receipt sync X** → 폰에서 읽음 → 태블릿 여전히 unread.
3. **Redis Pub/Sub 1 노드** → SPOF.
4. **같은 노드 + Redis 둘 다 send** → 중복.
5. **logout 시 device session 안 끊김** → 다른 user 도 메시지 봄.
6. **device id (FE UUID) 없음** → 같은 device 의 reconnect 인지 구분 X.
7. **session 영구 보관** → Redis 누적.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/multi-device-sync]]
- [[../implementation/multi-device-sync-impl]]
