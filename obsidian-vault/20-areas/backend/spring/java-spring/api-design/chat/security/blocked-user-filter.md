---
title: "Blocked user filter — 적용 위치"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:30:00+09:00
tags: [backend, java-spring, api-design, chat, security, block]
---

# Blocked user filter — 적용 위치

**[[security|↑ hub]]**

> 차단 검증은 **모든 message + presence + list query** 에 적용 필수.

---

## 1. 적용 위치 매트릭스

| 위치 | 검증 |
| --- | --- |
| 메시지 send | sender ↔ receiver 양쪽 block 확인 |
| 메시지 receive (broadcast) | recipient.blockedBy ⊃ sender.id ? skip |
| Room list (chat list) | DIRECT 의 상대 user 차단 시 표시 X |
| 그룹 내 메시지 | 그룹 멤버 중 차단 user 의 메시지 skip |
| Presence | 차단 user 의 presence 안 받음 |
| Typing indicator | 차단 user typing 안 받음 |
| 검색 | 차단 user 의 메시지 결과 제외 |
| 신규 그룹 초대 | 차단 user 가 초대 X |

---

## 2. 코드 (filter — 메시지 broadcast)

```java
@Service
@RequiredArgsConstructor
public class BlockFilter {

    private final BlockRepository blocks;
    private final RedisCache cache;

    /**
     * recipient 가 sender 를 차단했거나 sender 가 recipient 를 차단했으면 true.
     */
    public boolean isBlocked(UserId sender, UserId recipient) {
        var senderBlocked = cache.get(sender, k -> blocks.blockedByUser(k));
        var recipientBlocked = cache.get(recipient, k -> blocks.blockedByUser(k));
        return senderBlocked.contains(recipient) || recipientBlocked.contains(sender);
    }

    public Set<UserId> filterRecipients(UserId sender, Set<UserId> recipients) {
        return recipients.stream()
            .filter(r -> !isBlocked(sender, r))
            .collect(Collectors.toSet());
    }
}
```

→ Redis cache 5분 (block 변경 시 invalidate).

---

## 3. 함정

1. **메시지 send 만 검증** → 옛 메시지 broadcast 에 검증 X.
2. **단방향만** (sender → recipient) → 반대 방향 검증 누락.
3. **cache invalidate 안 함** → 차단 후 메시지 receive.
4. **그룹 멤버 list 에서 안 빠짐** → "그 사용자 있다" 표시.
5. **presence broadcast** 에 누락.

---

## 관련

- [[security|↑ hub]]
- [[../design-decisions/moderation-blocking]]
- [[../database/blocks-table]]
- [[../implementation/moderation-impl]]
