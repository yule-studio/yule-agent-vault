---
title: "chat design-decisions hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:55:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions]
---

# chat design-decisions hub

**[[../chat|↑ hub]]**

---

## 1. 결정 목록

| # | 결정 | 노트 |
| --- | --- | --- |
| 1 | **WebSocket vs SSE vs Polling** ★ | [[websocket-vs-sse-vs-polling]] |
| 2 | **STOMP vs raw WebSocket** ★ | [[stomp-vs-raw]] |
| 3 | Room types (DIRECT/GROUP/OPEN/SECRET) | [[room-types]] |
| 4 | Message storage (RDB vs Mongo vs Cassandra) | [[message-storage]] |
| 5 | **Message ordering** ★ | [[message-ordering]] |
| 6 | **Read receipt** ★ | [[read-receipt-strategy]] |
| 7 | **Multi-device sync** ★ | [[multi-device-sync]] |
| 8 | Presence | [[presence-strategy]] |
| 9 | Push fallback (offline) | [[push-fallback]] |
| 10 | Attachment | [[attachment-strategy]] |
| 11 | Search | [[search-strategy]] |
| 12 | Retention | [[retention-policy]] |
| 13 | Encryption (E2EE 옵션) | [[encryption-strategy]] |
| 14 | Moderation / Blocking | [[moderation-blocking]] |
| 15 | **Scale (Redis Pub/Sub)** ★ | [[scale-strategy]] |
| 16 | Kafka (F10+) | [[kafka-event-driven]] |

---

## 2. 결정 순서 (mermaid)

```mermaid
flowchart TD
    A[1. WebSocket] --> B[2. STOMP]
    B --> C[3. Room types]
    C --> D[4. Message storage]
    D --> E[5. Ordering]
    E --> F[6. Read receipt]
    F --> G[7. Multi-device]
    G --> H[15. Scale]
    H --> I[16. Kafka]
    C --> J[8. Presence]
    D --> K[9. Push fallback]
    D --> L[10. Attachment]
    D --> M[11. Search]
    D --> N[12. Retention]
    D --> O[13. Encryption]
    G --> P[14. Moderation]
```

---

## 3. 관련

- [[../chat|↑ hub]]
