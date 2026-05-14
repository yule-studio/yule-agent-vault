---
title: "Aggregate 경계"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:14:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - aggregate
---

# Aggregate 경계

**[[domain-model|↑ hub]]**

---

## 1. 본 모듈 경계

```mermaid
flowchart TB
    OD[다른 도메인<br/>signup / board / product / chat]
    OD -->|event| L[NotificationListener]
    L --> N[Notification aggregate<br/>outbox row]
    N -.->|by user_id| D[UserDevice aggregate]
    N -.->|by template_key| T[NotificationTemplate]
    N -->|fail max| DLQ
```

---

## 2. 같은 TX

- listener: outbox.save 만 (다른 aggregate 변경 X).
- worker: outbox + result UPDATE (같은 aggregate).
- device 등록: UserDevice 만.

---

## 3. eventual

- 다른 도메인 event → AFTER_COMMIT → outbox INSERT (다른 도메인 의 TX 가 commit 후).
- worker → 외부 채널 호출 (트랜잭션 밖).

---

## 4. 관련

- [[domain-model|↑ hub]]
- [[../transactions]]
- [[../architecture]]
