---
title: "실시간 채팅 (당근 스타일) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - chat
  - realtime
---

# 실시간 채팅 (당근 스타일) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 1:1 / 영속 / 읽음 / 푸시 fallback |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]] · [[websocket-stomp]] (전제).

---

## 1. 무엇을 만드는가

**1:1 채팅** (당근 스타일 — 거래 1건 = 채팅방 1개). 그룹채팅은 §10.

```
WebSocket (실시간):
  CONNECT /api/v1/ws + JWT
  SUBSCRIBE /topic/chat/{roomId}
  SEND /app/chat/{roomId}/send  { content }

REST (영속 / 폴링 백업):
  GET    /api/v1/chats                          # 내 채팅방 목록 (마지막 메시지 + 안 읽은 수)
  GET    /api/v1/chats/{roomId}/messages        # 메시지 페이지 (cursor)
  POST   /api/v1/chats/{roomId}/messages        # WebSocket fallback (HTTP send)
  POST   /api/v1/chats/{roomId}/read            # 읽음 처리
  POST   /api/v1/chats/with-user/{otherUserId}  # 채팅방 시작/조회 (idempotent)
```

### 1.1 비기능

- **모든 메시지 DB persist** (WebSocket 손실 시 폴링으로 복구)
- **읽음 처리** — last_read_message_id 모델
- **푸시 알림 fallback** — 상대 오프라인 / 인스턴스 disconnect 시 FCM
- **이미지 첨부** — [[file-upload-s3]] presigned URL
- **차단 / 신고** (별도)

---

## 2. 도메인 / DB

```sql
CREATE TABLE chat_rooms (
    id           CHAR(26) PRIMARY KEY,
    type         VARCHAR(20) NOT NULL,         -- DIRECT (1:1) / GROUP
    -- DIRECT 의 경우 user_a_id < user_b_id 정렬해서 unique
    user_a_id    CHAR(26),
    user_b_id    CHAR(26),
    last_message_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_chat_rooms_direct
    ON chat_rooms (user_a_id, user_b_id) WHERE type = 'DIRECT';
CREATE INDEX ix_chat_rooms_last_msg ON chat_rooms (last_message_at DESC);

CREATE TABLE chat_messages (
    id           CHAR(26) PRIMARY KEY,
    room_id      CHAR(26) NOT NULL REFERENCES chat_rooms(id) ON DELETE CASCADE,
    sender_id    CHAR(26) NOT NULL REFERENCES users(id),
    content      TEXT,
    image_url    VARCHAR(500),                 -- 별도 첨부
    deleted      BOOLEAN NOT NULL DEFAULT false,
    sent_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_chat_messages_room_sent ON chat_messages (room_id, sent_at DESC);

CREATE TABLE chat_room_members (
    room_id              CHAR(26) NOT NULL,
    user_id              CHAR(26) NOT NULL,
    last_read_message_id CHAR(26),
    joined_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at              TIMESTAMPTZ,
    PRIMARY KEY (room_id, user_id)
);
```

`user_a_id < user_b_id` 정렬 — DIRECT room 의 unique 보장. 두 사람이 어느 순서로 채팅 시작하든 같은 room.

---

## 3. UseCase

### 3.1 채팅방 시작 (idempotent)

```java
@Service
@RequiredArgsConstructor
public class StartDirectChatUseCase {

    private final ChatRoomRepository rooms;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public ChatRoomId handle(UserId currentUser, UserId otherUser) {
        if (currentUser.equals(otherUser))
            throw new BusinessException(ResponseCode.BAD_REQUEST, "자기 자신과 채팅 불가");

        // 정렬 (작은 ID 가 user_a)
        var a = currentUser.value().compareTo(otherUser.value()) < 0 ? currentUser : otherUser;
        var b = a == currentUser ? otherUser : currentUser;

        // 멱등 — 이미 있으면 그것 반환
        var existing = rooms.findDirectRoom(a, b);
        if (existing.isPresent()) return existing.get().id();

        var room = ChatRoom.createDirect(new ChatRoomId(ids.next()), a, b, Instant.now(clock));
        rooms.save(room);
        return room.id();
    }
}
```

### 3.2 메시지 전송

```java
@Service
@RequiredArgsConstructor
public class SendChatMessageUseCase {

    private final ChatRoomRepository rooms;
    private final ChatMessageRepository messages;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public ChatMessageId handle(ChatRoomId roomId, UserId senderId, String content,
                                String imageUrl) {
        var room = rooms.findById(roomId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "채팅방"));
        if (!room.isMember(senderId))
            throw new BusinessException(ResponseCode.FORBIDDEN);
        if ((content == null || content.isBlank()) && imageUrl == null)
            throw new BusinessException(ResponseCode.BAD_REQUEST, "내용 또는 이미지 필수");

        var now = Instant.now(clock);
        var msg = new ChatMessage(new ChatMessageId(ids.next()), roomId, senderId,
                                  content, imageUrl, now);
        messages.save(msg);
        room.updateLastMessageAt(now);
        rooms.save(room);

        // 이벤트 — STOMP broadcast + 푸시 알림 fallback
        events.publishEvent(new ChatMessageSent(roomId, msg.id(), senderId,
                                                room.otherMember(senderId),
                                                content, imageUrl, now));
        return msg.id();
    }
}

public record ChatMessageSent(
    ChatRoomId roomId, ChatMessageId messageId,
    UserId senderId, UserId receiverId,
    String content, String imageUrl, Instant sentAt
) implements DomainEvent { public Instant occurredAt() { return sentAt; } }
```

### 3.3 읽음 처리

```java
@Service
@RequiredArgsConstructor
public class MarkChatReadUseCase {

    private final ChatRoomMemberRepository members;
    private final ApplicationEventPublisher events;

    @Transactional
    public void handle(ChatRoomId roomId, UserId userId, ChatMessageId upToMessageId) {
        var member = members.findByRoomAndUser(roomId, userId)
            .orElseThrow(() -> new BusinessException(ResponseCode.FORBIDDEN));
        member.markRead(upToMessageId);
        members.save(member);

        events.publishEvent(new ChatRead(roomId, userId, upToMessageId, Instant.now()));
        // 상대에게 "읽음" 표시 STOMP broadcast
    }
}
```

---

## 4. STOMP broadcaster

```java
@Component
@RequiredArgsConstructor
public class ChatStompBroadcaster {

    private final SimpMessagingTemplate ws;
    private final PushNotificationService push;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onMessageSent(ChatMessageSent event) {
        var dto = new ChatMessageDto(
            event.messageId().value(), event.senderId().value(),
            event.content(), event.imageUrl(), event.sentAt()
        );

        // 1. 채팅방 전체 broadcast
        ws.convertAndSend("/topic/chat/" + event.roomId().value(), dto);

        // 2. 받는 사람의 unread badge 갱신
        ws.convertAndSendToUser(event.receiverId().value(), "/queue/chat-unread",
            new ChatUnreadEvent(event.roomId().value()));

        // 3. WebSocket 끊긴 사용자에게 푸시 fallback
        push.sendIfOffline(event.receiverId(),
            new PushPayload("새 메시지", event.content() != null
                ? event.content() : "[이미지]", Map.of("roomId", event.roomId().value())));
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onRead(ChatRead event) {
        ws.convertAndSend("/topic/chat/" + event.roomId().value() + "/read",
            new ChatReadDto(event.userId().value(), event.upToMessageId().value()));
    }
}
```

### 4.1 푸시 fallback

```java
@Service
@RequiredArgsConstructor
public class PushNotificationService {

    private final UserDeviceRepository devices;
    private final FcmClient fcm;
    private final WebSocketPresenceTracker presence;

    public void sendIfOffline(UserId receiverId, PushPayload payload) {
        if (presence.isOnline(receiverId)) return;        // WebSocket 연결 중 → 스킵

        devices.findActiveByUserId(receiverId).forEach(d -> {
            try {
                fcm.send(d.token(), payload);
            } catch (Exception e) {
                log.warn("FCM send failed for token={}", d.token(), e);
            }
        });
    }
}

// presence — STOMP connect/disconnect 이벤트로 추적
@Component
public class WebSocketPresenceTracker {

    private final RedisTemplate<String, String> redis;

    @EventListener
    public void onConnect(SessionConnectedEvent ev) {
        var userId = userIdFromEvent(ev);
        if (userId != null) redis.opsForValue().set("ws:online:" + userId, "1", Duration.ofMinutes(2));
    }

    @EventListener
    public void onDisconnect(SessionDisconnectEvent ev) {
        var userId = userIdFromEvent(ev);
        if (userId != null) redis.delete("ws:online:" + userId);
    }

    public boolean isOnline(UserId userId) {
        return Boolean.TRUE.equals(redis.hasKey("ws:online:" + userId.value()));
    }
}
```

---

## 5. REST endpoints

```java
@Tag(name = "채팅")
@RestController
@RequestMapping("/api/v1/chats")
@PreAuthorize("isAuthenticated()")
@RequiredArgsConstructor
public class ChatRestController {

    private final ChatQueryService query;
    private final StartDirectChatUseCase start;
    private final SendChatMessageUseCase send;
    private final MarkChatReadUseCase markRead;

    @GetMapping
    public ResponseEntity<CommonResponse<List<ChatRoomSummary>>> myRooms(Authentication auth) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            query.myRoomsWithUnread(new UserId(auth.getName())), "조회 성공"));
    }

    @GetMapping("/{roomId}/messages")
    public ResponseEntity<CommonResponse<MessageList>> messages(
        @PathVariable String roomId,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) int limit,
        Authentication auth
    ) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            query.messages(new ChatRoomId(roomId), new UserId(auth.getName()), cursor, limit),
            "조회 성공"));
    }

    @PostMapping("/with-user/{otherUserId}")
    public ResponseEntity<CommonResponse<Map<String, String>>> startWith(
        @PathVariable String otherUserId, Authentication auth
    ) {
        var roomId = start.handle(new UserId(auth.getName()), new UserId(otherUserId));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("roomId", roomId.value()), "채팅방 준비 완료"));
    }

    @PostMapping("/{roomId}/messages")
    public ResponseEntity<CommonResponse<Map<String, String>>> sendHttp(
        @PathVariable String roomId,
        @Valid @RequestBody SendMessageRequest req, Authentication auth
    ) {
        var messageId = send.handle(new ChatRoomId(roomId), new UserId(auth.getName()),
                                    req.content(), req.imageUrl());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("messageId", messageId.value()), "전송 완료"));
    }

    @PostMapping("/{roomId}/read")
    public ResponseEntity<CommonResponse<Void>> markRead(
        @PathVariable String roomId, @Valid @RequestBody MarkReadRequest req,
        Authentication auth
    ) {
        markRead.handle(new ChatRoomId(roomId), new UserId(auth.getName()),
            new ChatMessageId(req.upToMessageId()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "처리 완료"));
    }
}

public record SendMessageRequest(@Size(max = 4000) String content, String imageUrl) {}
public record MarkReadRequest(@NotBlank String upToMessageId) {}
```

### 5.1 unread 계산

```sql
SELECT count(*)
FROM chat_messages m
JOIN chat_room_members member ON member.room_id = m.room_id AND member.user_id = :userId
WHERE m.room_id = :roomId
  AND m.sender_id <> :userId
  AND (member.last_read_message_id IS NULL
       OR m.sent_at > (SELECT sent_at FROM chat_messages WHERE id = member.last_read_message_id));
```

→ 자주 호출이면 캐시. Redis ZSET 으로 메시지 list 보관도 옵션.

---

## 6. 이미지 첨부

```
1. [[file-upload-s3]] 로 presigned URL 받음
2. PUT 으로 S3 직접 업로드
3. POST /chats/{roomId}/messages { content: null, imageUrl: "https://cdn.../foo.jpg" }
```

→ 채팅 메시지는 imageUrl 만 저장 (S3 의 file 메타와 외래 연결 옵션).

---

## 7. 차단 / 신고 (간단)

```sql
CREATE TABLE user_blocks (
    blocker_id  CHAR(26) NOT NULL REFERENCES users(id),
    blocked_id  CHAR(26) NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id)
);

CREATE TABLE chat_reports (
    id           CHAR(26) PRIMARY KEY,
    reporter_id  CHAR(26) NOT NULL,
    reported_id  CHAR(26) NOT NULL,
    room_id      CHAR(26),
    reason       VARCHAR(50),                  -- SPAM / ABUSE / SCAM / OTHER
    detail       TEXT,
    status       VARCHAR(20) NOT NULL,         -- PENDING / REVIEWED / ACTION_TAKEN
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

차단되면:
- 송신 시 `UserBlockService.isBlocked(sender, receiver)` 확인 → 차단되면 메시지 거절 / 사일런트 (UX 정책)
- 차단된 사용자는 채팅방 목록에서 숨김

---

## 8. 함정 모음

### 함정 1 — DB persist 없이 STOMP 만
연결 끊긴 사이 메시지 손실. **모든 메시지 DB 저장 → STOMP broadcast** (DB 가 진실).

### 함정 2 — DIRECT 방 unique 누락
같은 두 사람이 여러 방. **(user_a_id, user_b_id) UNIQUE + 정렬**.

### 함정 3 — read 상태가 메시지마다 row
`message_id × user_id` 폭발. **last_read_message_id 만 멤버 row 에**.

### 함정 4 — 푸시 무조건 발송
WebSocket 연결 중인데도 푸시 = 중복 알림. **presence 확인**.

### 함정 5 — presence 가 죽으면 false positive
TTL 짧게 (1~2분) + heartbeat 가 갱신. WebSocket disconnect 도 명시적 evict.

### 함정 6 — 무한 채팅방 / 큰 메시지 list
오래된 메시지 archive (cold storage) 또는 cursor 페이지네이션.

### 함정 7 — 차단 무시
sender 가 blocked 인지 매번 검증. 캐시 (Redis SET) 권장.

### 함정 8 — 신고 처리 누락
moderation queue + 알람. 자동 ban (3회 신고) 정책 옵션.

### 함정 9 — typing indicator / read receipts 가 DB 부하
DB 저장 X. **STOMP broadcast 만** (휘발성 OK).

### 함정 10 — 1:1 → 그룹 마이그레이션
DIRECT 의 (user_a, user_b) 모델 → 멤버 N. 별도 schema. 처음부터 N 멤버 design 도 옵션.

---

## 9. 운영 체크리스트

- [ ] 메시지 DB persist + STOMP broadcast 분리
- [ ] WebSocket presence (Redis TTL 2m)
- [ ] 푸시 fallback (FCM / APNS)
- [ ] cursor 페이지네이션 (메시지 무한 스크롤)
- [ ] 차단 / 신고 운영 화면
- [ ] 이미지 첨부 — presigned URL
- [ ] last_read_message_id 만 (per-message status X)
- [ ] 오래된 메시지 archive 정책 (1년+)
- [ ] WebSocket connection 수 모니터

---

## 10. 그룹 채팅 (참고)

```sql
-- chat_rooms.type = 'GROUP', user_a_id / user_b_id 사용 X
-- chat_room_members 로 N 멤버 관리
-- 권한: 멤버만 send / subscribe
```

스키마 동일. unique 제약만 다름.

---

## 11. 관련

- [[websocket-stomp]] — 전제
- [[file-upload-s3]] — 이미지 첨부
- 푸시 알림 (notification — 추후)
- [[../common/security-config]] — STOMP CONNECT 인증
- [[api-design|↑ api-design hub]]
