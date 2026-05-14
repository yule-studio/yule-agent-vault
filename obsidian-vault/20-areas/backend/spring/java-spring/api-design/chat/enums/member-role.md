---
title: "MemberRole enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:16:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# MemberRole enum

**[[enums|↑ hub]]**

```java
public enum MemberRole {
    OWNER,     // 방장 (1명) — 방 삭제 / 방장 양도 가능
    ADMIN,     // 부방장 — 멤버 강퇴 / 모더
    MEMBER;    // 일반
}
```

| 권한 | OWNER | ADMIN | MEMBER |
| --- | --- | --- | --- |
| 메시지 send | O | O | O |
| 멤버 초대 | O | O | △ (정책) |
| 멤버 강퇴 | O | O | X |
| 방 이름 / 프로필 변경 | O | O | X |
| 방 삭제 | O | X | X |
| 방장 양도 | O | X | X |
| 메시지 hide (모더) | O | O | X |

→ DIRECT room: 역할 X (둘 다 동등).

---

## 관련

- [[enums|↑ hub]]
- [[../database/room-members-table]]
- [[../design-decisions/room-types]]
