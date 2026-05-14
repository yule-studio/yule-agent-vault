---
title: "Message fetch — REST cursor pagination"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:50:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, fetch]
---

# Message fetch — REST cursor pagination

**[[implementation|↑ hub]]**

---

## REST API

```http
GET /api/v1/rooms/{id}/messages?before=seq&limit=30

200 OK
{
  "data": [ ... ],
  "nextCursor": 12345,
  "hasMore": true
}
```

---

## 코드

```java
@RestController
@RequestMapping("/api/v1/rooms/{roomId}/messages")
@RequiredArgsConstructor
public class MessageController {

    private final MessageQueryService service;

    @GetMapping
    public PagedResponse<MessageDto> fetch(
            @PathVariable String roomId,
            @RequestParam(required = false) Long before,
            @RequestParam(defaultValue = "30") int limit,
            @AuthenticationPrincipal AuthUser auth) {
        var page = service.fetchBefore(
            RoomId.of(roomId), auth.userId(),
            before == null ? Long.MAX_VALUE : before,
            Math.min(limit, 100));
        return PagedResponse.of(page);
    }
}
```

---

## 함정

- 가입 안 한 room 의 메시지 접근 — service 에서 isMember 검증.
- limit 100+ → DoS.
- hidden 메시지 노출.
- 차단 user 의 메시지 노출 → blockedUserFilter.

---

## 관련

- [[implementation|↑ hub]]
- [[../database/messages-table]]
- [[../security/blocked-user-filter]]
