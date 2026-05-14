---
title: "Message send 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:48:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, message]
---

# Message send 구현 ★

**[[implementation|↑ hub]]**

---

## 1. STOMP Controller

```java
@Controller
@RequiredArgsConstructor
public class MessageStompController {

    private final MessageSendService service;

    @MessageMapping("/room/{roomId}/send")
    public void send(@DestinationVariable String roomId,
                     @Payload SendMessageRequest req,
                     Principal principal) {
        var user = UserId.of(principal.getName());
        service.send(RoomId.of(roomId), user, req);
    }
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class MessageSendService {

    private final RoomRepository rooms;
    private final RoomMemberRepository members;
    private final MessageRepository messages;
    private final BlockFilter blockFilter;
    private final MessageRateLimiter rateLimiter;
    private final SimpMessagingTemplate ws;
    private final MessagePublisher pub;       // Redis Pub/Sub
    private final PresenceService presence;
    private final NotificationPort notif;
    private final ChatHtmlSanitizer sanitizer;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public Message send(RoomId roomId, UserId sender, SendMessageRequest req) {
        // 1. rate limit
        require(rateLimiter.tryConsume(sender, roomId), "rate-limited");

        // 2. room + 멤버 검증
        var room = rooms.findById(roomId).orElseThrow();
        var senderMember = members.find(roomId, sender)
            .orElseThrow(() -> new ForbiddenException("not a member"));
        require(senderMember.leftAt() == null);

        // 3. seq INCR (Redis)
        var seq = messages.nextSeq(roomId);

        // 4. message create + sanitize
        var content = req.type() == MessageType.TEXT
            ? sanitizer.sanitize(req.content())
            : req.content();
        var msg = Message.create(
            MessageId.next(), roomId, sender, seq,
            req.type(), content, req.metadata(),
            req.replyToId(), req.clientMessageId(), clock.now());
        messages.save(msg);

        // 5. room.last_message 갱신
        room.updateLastMessage(msg.id(), msg.createdAt());
        rooms.save(room);

        // 6. AFTER_COMMIT → broadcast + push fallback
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                public void afterCommit() {
                    broadcastAndPush(room, msg);
                }
            });
        return msg;
    }

    private void broadcastAndPush(Room room, Message msg) {
        // 같은 노드 broadcast
        ws.convertAndSend("/topic/room/" + msg.roomId().value(), msg.toDto());

        // cross-node — Redis publish
        pub.publishToRoom(msg.roomId(), msg.toDto());

        // offline 사용자에게 push
        var memberList = members.findActiveMembers(msg.roomId());
        for (var m : memberList) {
            if (m.userId().equals(msg.senderId())) continue;
            if (blockFilter.isBlocked(msg.senderId(), m.userId())) continue;
            if (presence.isOnline(m.userId())) continue;     // 이미 broadcast 받음
            notif.chatMessageOffline(m.userId(),
                new ChatMessageNotif(msg.roomId(), room.name(), msg.toPreview()));
        }
    }
}
```

---

## 3. 함정

- broadcast 가 트랜잭션 안 → DB 락.
- send 후 fail (broadcast 실패) → 사용자 화면에 메시지 없음 → reload 시 받음 OK.
- Redis Pub/Sub 만 broadcast → 같은 노드 의 user 못 받음 (이중 send 권장).

자세히: [[../transactions]] · [[../design-decisions/multi-device-sync]].

---

## 4. 관련

- [[implementation|↑ hub]]
- [[connect-auth-impl]]
- [[read-receipt-impl]]
- [[multi-device-sync-impl]]
- [[../domain-model/message-aggregate]]
