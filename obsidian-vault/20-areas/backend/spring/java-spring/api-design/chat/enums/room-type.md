---
title: "RoomType enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:10:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# RoomType enum

**[[enums|↑ hub]]**

```java
public enum RoomType {
    DIRECT(2),      // 1:1 (두 user pair)
    GROUP(500),     // 일반 그룹 (방장 + 초대제)
    OPEN(1500),    // 오픈채팅 (검색 / 누구나)
    SECRET(100);   // 비밀 + E2EE 옵션

    private final int defaultMaxMembers;
    RoomType(int max) { this.defaultMaxMembers = max; }
}
```

자세히: [[../design-decisions/room-types]].

---

## 관련

- [[enums|↑ hub]]
- [[../design-decisions/room-types]]
