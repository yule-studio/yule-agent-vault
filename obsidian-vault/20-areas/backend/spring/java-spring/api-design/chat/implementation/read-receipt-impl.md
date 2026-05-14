---
title: "Read receipt 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:52:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, read]
---

# Read receipt 구현 ★

**[[implementation|↑ hub]]**

---

## 1. STOMP

```java
@MessageMapping("/room/{roomId}/read")
public void read(@DestinationVariable String roomId,
                 @Payload ReadReceiptRequest req,
                 Principal principal) {
    readService.markRead(RoomId.of(roomId),
        UserId.of(principal.getName()),
        req.lastSeq());
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class ReadReceiptService {

    private final RoomMemberRepository members;
    private final SimpMessagingTemplate ws;
    private final MessagePublisher pub;
    private final Clock clock;

    @Transactional
    public void markRead(RoomId roomId, UserId user, long lastSeq) {
        var member = members.find(roomId, user).orElseThrow();
        if (lastSeq <= member.lastReadSeq()) return;        // 감소 X

        member.updateLastReadSeq(lastSeq, clock.now());
        members.save(member);

        // broadcast — debounce 1s (1번 발송 / 1초)
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                public void afterCommit() {
                    var payload = new ReadEventDto(roomId.value(), user.value(), lastSeq);
                    ws.convertAndSend("/topic/room/" + roomId.value() + "/read", payload);
                    pub.publishToRoom(roomId, payload);
                }
            });
    }
}
```

---

## 3. 함정

- last_read_seq 감소 (사용자 스크롤 업) → MAX(old, new) 만 update.
- 그룹 의 broadcast 폭주 → debounce 1초 (서버 측에서 buffer).
- 멀티 디바이스 동시 read INSERT — UNIQUE PK + ON CONFLICT.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/read-receipt-strategy]]
- [[../database/message-reads-table]]
