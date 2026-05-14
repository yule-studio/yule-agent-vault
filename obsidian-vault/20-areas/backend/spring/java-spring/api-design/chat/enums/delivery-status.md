---
title: "DeliveryStatus enum (메시지 전달 상태)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:20:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# DeliveryStatus enum (메시지 전달 상태)

**[[enums|↑ hub]]**

```java
public enum DeliveryStatus {
    PENDING,     // FE 가 send 시도 중 (network 대기)
    SENT,        // 서버 ACK 받음 (DB INSERT 완료)
    DELIVERED,   // 1+ recipient device 도착
    READ,        // 1+ recipient 가 읽음
    FAILED;      // FE network fail 또는 server reject
}
```

→ MessageStatus 와 다름: 본 enum 은 **FE 측 의 상태** (각 메시지 의 표시 — 카톡 의 ⌛ → ✓ → ✓✓).

---

## 관련

- [[enums|↑ hub]]
- [[message-status]]
