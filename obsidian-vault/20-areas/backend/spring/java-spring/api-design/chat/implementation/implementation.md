---
title: "chat implementation hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:40:00+09:00
tags: [backend, java-spring, api-design, chat, implementation]
---

# chat implementation hub

**[[../chat|↑ hub]]**

---

| 노트 | 영역 |
| --- | --- |
| [[websocket-stomp-config]] ★ | STOMP broker + Spring config |
| [[connect-auth-impl]] ★ | JWT in CONNECT frame |
| [[room-impl]] | Room CRUD |
| [[message-send-impl]] ★ | 메시지 send + persist + broadcast |
| [[message-fetch-impl]] | REST 페이징 |
| [[read-receipt-impl]] ★ | last_read_seq |
| [[multi-device-sync-impl]] ★ | Redis Pub/Sub fan-out |
| [[presence-impl]] | heartbeat / typing |
| [[attachment-impl]] | S3 presigned |
| [[search-impl]] | FTS |
| [[moderation-impl]] | 신고 / block |
| [[push-fallback-impl]] | offline → notification |
| [[google-login-impl]] ★ | OAuth2 (Project 4 요구) |
| [[kafka-integration]] | F10+ |

---

## 관련

- [[../chat|↑ hub]]
- [[../architecture]]
