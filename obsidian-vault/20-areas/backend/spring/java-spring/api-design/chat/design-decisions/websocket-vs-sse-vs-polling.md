---
title: "Transport — WebSocket vs SSE vs Polling"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:58:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, websocket, transport]
---

# Transport — WebSocket vs SSE vs Polling ★

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — WebSocket

---

## 2. 비교 표

| 항목 | **WebSocket** ★ | SSE (Server-Sent Events) | Long polling | Short polling |
| --- | --- | --- | --- | --- |
| 방향 | 양방향 (full duplex) | server → client only | server → client (request-response) | request-response |
| 연결 | 지속 (1 TCP) | 지속 (HTTP) | 1 request 당 ~30s | 매 N 초 |
| 브라우저 호환 | 96%+ | 96%+ | 100% | 100% |
| 모바일 | 가능 | 가능 (Android 일부 X) | 가능 | 가능 |
| HTTP/2 | 별도 spec | O | O | O |
| 프록시 친화 | nginx 설정 필요 | OK | OK | OK |
| latency (메시지 도착) | < 100ms | < 100ms | ~ poll 주기 | ~ poll 주기 |
| typing indicator | 가능 | server only | X | X |
| 양방향 message | O | client 가 별도 REST 호출 | client 가 별도 REST | client 가 별도 REST |
| 연결 수 (server) | ~1만 / 노드 | ~1만 / 노드 | ~ 1만 / 노드 | 무관 (request-response) |
| 메시지 별도 ACK | application 구현 | application | HTTP response | HTTP response |
| 인증 (CONNECT) | header / first frame | header / cookie | 매 요청 | 매 요청 |

---

## 3. 왜 WebSocket (SSE / polling 아님)

### 3.1 왜 필요

- **chat 의 양방향**: send + receive 둘 다 — WebSocket 1 연결로.
- **typing indicator**: client → server 도 발송 — SSE 불가능.
- **latency**: 카톡 < 1s 표준 — polling 으로 못 맞춤.
- **연결 수 효율**: 1 TCP / user (multi-room 까지 sub).

### 3.2 안 하면 어떤 문제

| 선택 | 문제 |
| --- | --- |
| SSE | 양방향 X — 매 send 마다 REST → 부하 ↑ |
| Long polling | 30s 마다 reconnect → 30s 메시지 지연 가능 |
| Short polling 1s | 서버 부하 ↑↑ + 1s 지연 |
| 매번 polling | 모바일 battery 영향 |

### 3.3 트레이드오프

- WebSocket = 양방향 + low latency + 프록시 설정 (Upgrade header) + sticky session 필요.
- SSE = 단순 + server → client only.
- Polling = 가장 단순 + 매번 인증 가능 + latency / 부하 ↑.

---

## 4. 다른 컨텍스트

| 비즈니스 | 선택 | 이유 |
| --- | --- | --- |
| **카톡 / 라인** ★ | WebSocket | 본 vault |
| 트위치 채팅 | WebSocket | 양방향 + 대량 |
| 알림 (chat 아님) | SSE | 1방향만 |
| 주식 시세 | SSE 또는 WebSocket | 1방향 (price feed) |
| 단순 알림 (Slack desktop) | WebSocket (구식 Slack 은 polling) | 양방향 |
| Github actions log streaming | SSE | 1방향 |

---

## 5. 함정

1. **HTTP/1.1 무한 idle 없이 사용** → Upgrade 안 됨.
2. **load balancer sticky session 없음** → 매번 다른 노드.
3. **HTTPS / WSS** 없이 production → MITM.
4. **proxy timeout** (nginx 60s default) → 연결 끊김.
5. **CORS / Origin 무시** → CSRF.

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[stomp-vs-raw]]
- [[scale-strategy]]
- [[../implementation/websocket-stomp-config]]
- [[../../websocket-stomp|↗ websocket-stomp recipe (기술)]]
