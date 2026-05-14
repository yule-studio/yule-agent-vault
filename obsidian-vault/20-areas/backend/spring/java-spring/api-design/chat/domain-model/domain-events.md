---
title: "chat 도메인 이벤트"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:02:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, event]
---

# chat 도메인 이벤트

**[[domain-model|↑ hub]]**

---

| Event | 트리거 | 처리 |
| --- | --- | --- |
| MessageSent | 메시지 send | WebSocket broadcast + push fallback |
| MessageEdited | 5분 안 편집 | broadcast |
| MessageHardDeleted | 5분 안 삭제 | broadcast (UI 제거) |
| MessageSoftDeleted | 5분 후 삭제 | broadcast (UI hidden) |
| MessageHidden | admin 모더 | broadcast |
| MessageRead | 읽음 표시 | broadcast 읽음 카운트 |
| RoomCreated | 방 생성 | broadcast member 들에게 |
| RoomMemberJoined | 입장 | system 메시지 + broadcast |
| RoomMemberLeft | 퇴장 | system 메시지 |
| RoomMemberKicked | 강퇴 | system 메시지 + 강퇴자 disconnect |
| TypingStarted / TypingStopped | client → server | broadcast (debounce) |
| PresenceChanged | online/away/offline | friends 에게 broadcast |
| ChatSessionConnected / Disconnected | WebSocket | session tracking |
| UserBlocked / Unblocked | block | 사용자 cache invalidate |

---

## 관련

- [[domain-model|↑ hub]]
- [[../implementation/message-send-impl]]
- [[../design-decisions/multi-device-sync]]
