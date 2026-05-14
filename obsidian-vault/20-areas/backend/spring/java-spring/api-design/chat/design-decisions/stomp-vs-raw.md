---
title: "STOMP vs raw WebSocket"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, stomp]
---

# STOMP vs raw WebSocket ★

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — STOMP

---

## 2. 비교

| 항목 | **STOMP** ★ | raw WebSocket | Socket.IO |
| --- | --- | --- | --- |
| Sub-protocol | text-based frame | binary / text | 자체 |
| Pub/Sub destination | `/topic/x` / `/user/queue/x` | 직접 구현 | `socket.emit("event")` |
| Spring 통합 | `@MessageMapping`, `convertAndSendToUser` | 직접 구현 | 비공식 |
| Broker | SimpleBroker / RabbitMQ / ActiveMQ | 직접 | 자체 |
| CONNECT auth | nativeHeaders + interceptor | URL query / first frame | handshake |
| 표준 | RFC-like (STOMP 1.2) | RFC 6455 | 비표준 |
| 모바일 client lib | java-stomp / iOS StompKit | OkHttp / Starscream | socket.io-client |
| FE 통합 | stomp.js (sockjs-client) | native WebSocket | socket.io 강제 |
| Heartbeat | STOMP frame | ping/pong | 자체 |

---

## 3. 왜 STOMP

### 3.1 왜 필요

- **destination 의미**: `/topic/room/X` (group) vs `/user/queue/Y` (1:1) — chat 의 자연 model.
- **Spring 통합**: `@MessageMapping("/room/{id}/send")` + `convertAndSendToUser`.
- **RabbitMQ relay** (F5+): broker 만 변경, application 코드 그대로.

### 3.2 안 하면 어떤 문제

- raw WebSocket: route / dispatch 직접 구현 — 매번 JSON parse + 분기.
- Socket.IO: Spring 비공식 + non-standard — multi-stack 어려움.

### 3.3 트레이드오프

- STOMP: 표준 + Spring + scale-out (RabbitMQ) + 약간의 overhead (frame format).
- raw: 가장 빠름 + 모든 거 직접.
- Socket.IO: easy + lock-in.

---

## 4. 본 vault STOMP 구조

```
CONNECT (with JWT in headers)
  ↓ StompAuthInterceptor 검증
SUBSCRIBE /topic/room/{id}
  ↓ broker
SUBSCRIBE /user/queue/notif
  ↓ user-specific (멀티 디바이스 fan-out)
SEND /app/room/{id}/send
  ↓ @MessageMapping
  → service → DB INSERT
  → convertAndSend "/topic/room/{id}"
DISCONNECT
  ↓ event listener → cleanup session / presence
```

---

## 5. 함정

1. **CONNECT 후 auth 없음** — interceptor 누락 시 anonymous 사용자.
2. **/topic/** vs `/queue/` 혼동 — topic = broadcast, queue = 1:1.
3. **broker 의 max subscription** (SimpleBroker 의 in-memory) — F5+ RabbitMQ.
4. **server → client send 시 user 가 connect 안 됨** — `SimpUserRegistry` 로 check.

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[websocket-vs-sse-vs-polling]]
- [[../implementation/websocket-stomp-config]]
- [[../../websocket-stomp|↗ websocket-stomp recipe]]
