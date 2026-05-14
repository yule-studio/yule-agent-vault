---
title: "chat overview — end-to-end"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:43:00+09:00
tags: [backend, java-spring, api-design, chat, overview]
---

# chat overview — end-to-end

**[[chat|↑ hub]]**

---

## 1. 큰 그림

```mermaid
flowchart TB
    subgraph user["사용자"]
        U1[사용자 A<br/>폰]
        U2[사용자 A<br/>태블릿]
        U3[사용자 B<br/>폰]
    end

    subgraph fe["FE"]
        F1[stomp.js<br/>WebSocket client]
    end

    subgraph server["서버"]
        WS[WebSocket Gateway<br/>STOMP broker]
        AUTH[JWT 검증<br/>CONNECT frame]
        ROOM[Room service]
        MSG[Message service]
        READ[ReadReceipt service]
        REDIS[Redis Pub/Sub<br/>cross-node]
    end

    subgraph store["저장"]
        DB[(PostgreSQL<br/>messages / rooms / reads)]
        S3[S3<br/>attachments]
    end

    subgraph notif["외부"]
        N[notification<br/>offline push fallback]
    end

    U1 --> F1
    U2 --> F1
    U3 --> F1
    F1 -->|CONNECT + JWT| WS
    WS --> AUTH
    F1 -->|SEND /app/room/X/send| MSG
    MSG --> DB
    MSG --> REDIS
    REDIS -->|cross-node fan-out| WS
    WS -->|/topic/room/X| F1
    MSG -->|offline user| N
    F1 -->|attachment URL| S3
```

자세히: [[architecture]].

---

## 2. 시퀀스 — 1:1 메시지 발송 (single device)

```mermaid
sequenceDiagram
    autonumber
    participant A as User A
    participant FE as A's FE (stomp.js)
    participant WS as WebSocket
    participant Server
    participant DB
    participant B as User B (online)
    participant FE_B as B's FE

    A->>FE: 메시지 입력 "안녕"
    FE->>WS: SEND /app/room/r1/send { content: "안녕" }
    WS->>Server: handle SEND
    Server->>DB: messages INSERT (room=r1, sender=A, seq=N)
    Server->>WS: convertAndSend /topic/room/r1
    WS->>FE_B: STOMP MESSAGE frame
    FE_B-->>B: 메시지 표시

    Note over Server: read receipt
    B->>FE_B: 메시지 화면 보임
    FE_B->>WS: SEND /app/room/r1/read { messageId: M }
    WS->>Server: handle
    Server->>DB: message_reads INSERT (msg=M, user=B)
    Server->>WS: convertAndSend /topic/room/r1/read
    WS->>FE: B read M (1 → 0)
```

---

## 3. 시퀀스 — 그룹 + offline 사용자 (push fallback)

```mermaid
sequenceDiagram
    participant A as User A
    participant Server
    participant DB
    participant B as User B (online)
    participant C as User C (offline)
    participant Notif as notification

    A->>Server: SEND /app/room/group1/send
    Server->>DB: messages INSERT
    Server->>Server: room.members = [A, B, C]
    Server->>Server: check presence

    par online
        Server->>B: STOMP MESSAGE
    and offline
        Server->>Notif: outbox INSERT (CHAT_MESSAGE → C)
        Notif->>C's device: FCM push
    end

    Note over C: app open
    C->>Server: GET /api/v1/rooms/group1/messages?cursor=last
    Server-->>C: 안 본 메시지들
```

자세히: [[design-decisions/push-fallback]].

---

## 4. 시퀀스 — 멀티 디바이스 동기화 ★

```mermaid
sequenceDiagram
    participant A1 as A (폰)
    participant A2 as A (태블릿)
    participant Server
    participant Redis as Redis Pub/Sub
    participant DB

    Note over A1,A2: 둘 다 같은 user A, 다른 WebSocket session

    A1->>Server: SEND /app/room/r1/send "안녕"
    Server->>DB: messages INSERT
    Server->>Redis: publish "room:r1" + message

    par 같은 노드
        Server->>A1: /topic/room/r1 (자기 메시지 echo)
    and 다른 노드 (subscribe room:r1)
        Redis-->>Server: receive
        Server->>A2: /topic/room/r1 (다른 디바이스 sync)
    end

    Note over A1,A2: A 가 어느 디바이스로 보내도 모든 디바이스 sync
```

자세히: [[design-decisions/multi-device-sync]] · [[implementation/multi-device-sync-impl]].

---

## 5. 상태 머신 (메시지)

```mermaid
stateDiagram-v2
    [*] --> SENT: server INSERT 완료
    SENT --> DELIVERED: 1+ device 받음
    DELIVERED --> READ: 사용자 봄
    SENT --> DELETED: 본인 삭제 (5분 안)
    READ --> HIDDEN: admin 모더
```

자세히: [[enums/message-status]].

---

## 6. ERD 큰 그림

```mermaid
erDiagram
    USERS ||--o{ ROOM_MEMBERS : joins
    ROOMS ||--|{ ROOM_MEMBERS : has
    ROOMS ||--o{ MESSAGES : contains
    MESSAGES ||--o{ MESSAGE_READS : tracked-by
    USERS ||--o{ MESSAGE_READS : reads
    USERS ||--o{ CHAT_SESSIONS : connects
    MESSAGES ||--o{ ATTACHMENTS : has
    USERS ||--o{ BLOCKS : blocks
```

자세히: [[database/database]].

---

## 7. 관련

- [[chat|↑ hub]]
- [[prerequisites]] · [[requirements]] · [[architecture]] · [[transactions]] · [[implementation-order]]
