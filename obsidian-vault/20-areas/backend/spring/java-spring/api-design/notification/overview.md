---
title: "notification overview — end-to-end"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - overview
---

# notification overview — end-to-end

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[notification|↑ hub]]**

---

## 1. 큰 그림

```mermaid
flowchart TB
    subgraph trigger["트리거 (도메인)"]
        T1[OrderPaid]
        T2[CommentCreated]
        T3[ChatMessage]
        T4[PaymentFailed]
        T5[SecurityAlert<br/>강제 ON]
    end

    subgraph listener["Listener (도메인별)"]
        L1[OrderListener]
        L2[BoardListener]
        L3[ChatListener]
    end

    subgraph notif["notification 모듈"]
        P[Preference 검증]
        D[Dedup check]
        O[outbox INSERT]
        W[Worker findPending]
        R[ChannelRouter]
        TPL[Template 렌더]
    end

    subgraph channels["채널 adapter"]
        FCM[FCM<br/>Android]
        APNs[APNs<br/>iOS]
        WP[WebPush<br/>Web]
        SES[AWS SES<br/>Email]
        SL[Slack<br/>Admin]
        IA[In-App<br/>DB row]
    end

    subgraph external["외부"]
        F[Firebase]
        A[Apple Push]
        AWS[AWS SES]
        S[Slack API]
    end

    trigger --> listener
    listener --> P
    P --> D
    D --> O
    O --> W
    W --> R
    R --> TPL
    TPL --> FCM
    TPL --> APNs
    TPL --> WP
    TPL --> SES
    TPL --> SL
    TPL --> IA
    FCM --> F
    APNs --> A
    SES --> AWS
    SL --> S
```

---

## 2. 시퀀스 — 단일 알림 발송

```mermaid
sequenceDiagram
    autonumber
    participant App as 도메인
    participant L as Listener
    participant DB
    participant W as Worker
    participant R as ChannelRouter
    participant FCM
    participant User as 사용자

    Note over App,DB: @Transactional
    App->>DB: 비즈니스 INSERT/UPDATE
    App->>L: publishEvent(OrderPaid)
    DB-->>App: COMMIT

    rect rgb(219, 234, 254)
    note over L: AFTER_COMMIT
    L->>DB: user_preference 확인
    alt 사용자 OFF
        L->>L: skip
    else 사용자 ON
        L->>DB: notification_outbox INSERT (event_id UNIQUE)
        L->>DB: in-app notifications INSERT (옵션)
    end
    end

    Note over W: @Scheduled fixedDelay 1s
    W->>DB: SELECT FOR UPDATE SKIP LOCKED
    W->>DB: status=PROCESSING + attempts++
    W->>R: route(notification)
    R->>DB: user_devices 조회 (active)
    loop per device
        R->>FCM: send(token, payload)
        FCM-->>R: ack / token-invalid
    end
    R->>DB: 결과 기록
    W->>DB: status=SENT or FAILED
    Note over W: 실패 시 exp backoff retry
```

---

## 3. 흐름 — 핵심 6단계

```
1. 트리거 (도메인 이벤트)
2. Listener (AFTER_COMMIT) — preference 검증 + outbox INSERT
3. Worker pickup — SKIP LOCKED
4. ChannelRouter — platform / channel 결정
5. Template 렌더 — i18n + 변수 치환
6. 채널 adapter — FCM / APNs / SES / WebPush / Slack / in-app
   ↓
   성공 → SENT
   일시 실패 → exp backoff retry
   영구 실패 (invalid token / bad address) → DLQ + admin
```

---

## 4. ERD 큰 그림

```mermaid
erDiagram
    USERS ||--o{ USER_DEVICES : owns
    USERS ||--|| USER_NOTIFICATION_PREFERENCES : has
    USERS ||--o{ NOTIFICATION_OUTBOX : receives
    USERS ||--o{ NOTIFICATIONS : in-app
    NOTIFICATION_OUTBOX ||--o{ NOTIFICATION_DLQ : moves-on-fail
    NOTIFICATION_OUTBOX }o--|| NOTIFICATION_TEMPLATES : uses
```

자세히: [[database/database]].

---

## 5. 상태 머신 (outbox row)

```mermaid
stateDiagram-v2
    [*] --> PENDING: listener INSERT
    PENDING --> PROCESSING: worker pickup
    PROCESSING --> SENT: 채널 ack
    PROCESSING --> PENDING: transient 실패 (retry)
    PROCESSING --> FAILED: permanent (invalid token)
    PENDING --> DLQ: max retry 초과
    FAILED --> DLQ
    DLQ --> PENDING: admin replay
```

자세히: [[enums/notification-status]].

---

## 6. 관련

- [[notification|↑ hub]]
- [[prerequisites]] — 시작 전
- [[requirements]] — 35 AC
- [[architecture]] — Hexagonal + ChannelRouter
- [[transactions]]
- [[implementation-order]]
