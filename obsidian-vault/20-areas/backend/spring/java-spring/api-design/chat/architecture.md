---
title: "chat architecture — Hexagonal + WebSocket layer"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:50:00+09:00
tags: [backend, java-spring, api-design, chat, architecture]
---

# chat architecture — Hexagonal + WebSocket layer

**[[chat|↑ hub]]**

---

## 1. 패키지 구조

```
com.example.chat
├── domain/
│   ├── room/
│   │   ├── Room.java                 ★ aggregate
│   │   ├── RoomMember.java
│   │   ├── RoomType.java
│   │   └── RoomRepository.java
│   ├── message/
│   │   ├── Message.java               ★ aggregate
│   │   ├── MessageId.java
│   │   ├── MessageType.java
│   │   ├── MessageStatus.java
│   │   ├── MessageRead.java
│   │   └── MessageRepository.java
│   ├── session/
│   │   ├── ChatSession.java           ← WebSocket session
│   │   └── ChatSessionRepository.java
│   ├── presence/
│   │   ├── Presence.java
│   │   └── PresenceRepository.java
│   └── port/
│       ├── MessagePublisher.java      ← cross-node fan-out port
│       ├── NotificationPort.java      ← offline push (notification 모듈)
│       └── AttachmentStorage.java
│
├── application/
│   ├── ChatService.java
│   ├── RoomService.java
│   ├── MessageSendService.java        ★
│   ├── ReadReceiptService.java        ★
│   ├── PresenceService.java
│   ├── MultiDeviceSyncService.java    ★
│   └── ChatMessageListener.java       ← Redis Pub/Sub subscriber
│
├── infrastructure/
│   ├── websocket/
│   │   ├── WebSocketConfig.java       ★ STOMP broker
│   │   ├── StompAuthInterceptor.java  ★ CONNECT JWT 검증
│   │   ├── WebSocketEventListener.java ← session connect/disconnect
│   │   └── SimpUserRegistryAdapter.java
│   ├── pubsub/
│   │   └── RedisMessagePublisher.java ← cross-node
│   ├── persistence/jpa/...
│   ├── attachment/
│   │   └── S3AttachmentStorage.java
│   └── notification/
│       └── NotificationModuleAdapter.java ← notification port 구현
│
├── interfaces/
│   ├── stomp/                          ← @MessageMapping
│   │   ├── MessageStompController.java ★
│   │   ├── ReadReceiptStompController.java
│   │   ├── TypingStompController.java
│   │   └── PresenceStompController.java
│   ├── api/                            ← REST
│   │   ├── RoomController.java
│   │   ├── MessageController.java     ← GET / 페이징
│   │   ├── AttachmentController.java   ← presigned URL
│   │   └── DeviceController.java
│   └── admin/
│       └── AdminChatController.java
│
└── config/
    ├── SecurityConfig.java
    ├── RedisPubSubConfig.java
    └── ChatProperties.java
```

---

## 2. WebSocket Layer (STOMP)

```mermaid
flowchart TB
    Client[FE stomp.js] -->|CONNECT + JWT| Auth[StompAuthInterceptor]
    Auth -->|valid| Broker[STOMP Broker<br/>SimpleBroker / RabbitMQ]
    Client -->|SUBSCRIBE /topic/room/X| Broker
    Client -->|SEND /app/room/X/send| Controller[@MessageMapping]
    Controller --> Service[MessageSendService]
    Service --> DB[(PostgreSQL)]
    Service --> Pub[RedisMessagePublisher]
    Pub -->|publish "room:X"| Redis[(Redis Pub/Sub)]
    Redis -.->|fan-out| OtherNodes[다른 노드]
    Service -->|/topic/room/X| Broker
    Broker --> Client
```

---

## 3. Hexagonal port

| Port | 구현 | 책임 |
| --- | --- | --- |
| `MessagePublisher` | `RedisMessagePublisher` | cross-node 메시지 broadcast |
| `NotificationPort` | `NotificationModuleAdapter` | offline 사용자 push (notification 모듈 호출) |
| `AttachmentStorage` | `S3AttachmentStorage` | 첨부 presigned URL |
| `MessageRepository` | `JpaMessageRepository` | DB |
| `ChatSessionRepository` | `JpaChatSessionRepository` | WebSocket session list |
| `PresenceRepository` | `RedisPresenceRepository` | presence (Redis) |

---

## 4. STOMP routing

| Destination | 방향 | 용도 |
| --- | --- | --- |
| `/app/room/{id}/send` | client → server | 메시지 발송 |
| `/app/room/{id}/read` | client → server | 읽음 표시 |
| `/app/room/{id}/typing` | client → server | typing indicator |
| `/topic/room/{id}` | server → client | room 의 모든 member |
| `/topic/room/{id}/read` | server → client | 읽음 broadcast |
| `/topic/room/{id}/typing` | server → client | typing |
| `/user/queue/notif` | server → 특정 user | 1:1 push (멀티 디바이스 모든 session) |
| `/user/queue/presence` | server → user | 친구 presence 변경 |

→ `/user/queue/*` = Spring 의 user destination (`SimpUserRegistry` + `convertAndSendToUser`).

자세히: [[implementation/websocket-stomp-config]].

---

## 5. 모듈 분리 (F12+)

```
F0~F8: 단일 monolith (com.example.app)
F10+: chat 분리 옵션 (com.example.chat-service)
     ↓
     Kafka 통해 notification 통신
```

자세히: [[design-decisions/kafka-event-driven]].

---

## 6. 관련

- [[chat|↑ hub]]
- [[transactions]]
- [[domain-model/domain-model]]
- [[implementation/websocket-stomp-config]]
- [[design-decisions/scale-strategy]]
