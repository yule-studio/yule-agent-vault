---
title: "Repository ports + adapter ports"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:04:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, port]
---

# Repository ports + adapter ports

**[[domain-model|↑ hub]]**

---

## 1. Repositories

```java
public interface RoomRepository {
    Room save(Room r);
    Optional<Room> findById(RoomId id);
    Optional<Room> findDirectPair(UserId a, UserId b);
    List<RoomMember> findActiveMembers(RoomId id);
    List<Room> findRoomsForUser(UserId user, Pageable page);
}

public interface MessageRepository {
    Message save(Message m);
    Optional<Message> findById(MessageId id);
    List<Message> findByRoomCursor(RoomId id, long beforeSeq, int limit);
    long nextSeq(RoomId id);              // Redis INCR
}

public interface MessageReadRepository {
    void recordRead(RoomId room, UserId user, long lastReadSeq, Instant now);
    Optional<Long> findLastReadSeq(RoomId room, UserId user);
}

public interface ChatSessionRepository {        // Redis
    void register(ChatSession s);
    void unregister(SessionId id);
    List<ChatSession> findByUser(UserId user);
    void heartbeat(SessionId id, Instant now);
}

public interface PresenceRepository {           // Redis
    void update(UserId user, PresenceStatus status, Instant now);
    Map<UserId, PresenceStatus> findMany(Collection<UserId> users);
}

public interface BlockRepository {
    void block(UserId blocker, UserId blocked, String reason, Instant now);
    void unblock(UserId blocker, UserId blocked);
    Set<UserId> blockedByUser(UserId blocker);  // Redis cache
}
```

---

## 2. Adapter ports

```java
public interface MessagePublisher {           // cross-node Redis Pub/Sub
    void publishToRoom(RoomId room, Object event);
    void publishToUser(UserId user, Object event);
}

public interface NotificationPort {           // notification 모듈 호출
    void chatMessageOffline(UserId recipient, ChatMessageNotif payload);
}

public interface AttachmentStorage {
    String presignedPutUrl(String key, String mime, long maxSize);
    String publicUrl(String key);
}
```

---

## 관련

- [[domain-model|↑ hub]]
- [[../architecture]]
- [[../implementation/message-send-impl]]
