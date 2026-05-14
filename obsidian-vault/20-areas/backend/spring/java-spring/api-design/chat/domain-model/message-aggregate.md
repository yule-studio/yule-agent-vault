---
title: "Message Aggregate ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:54:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, aggregate, message]
---

# Message Aggregate ★

**[[domain-model|↑ hub]]**

---

```java
public final class Message {
    private final MessageId id;
    private final RoomId roomId;
    private final UserId senderId;
    private long seq;
    private final MessageType type;
    private String content;
    private final Map<String, Object> metadata;
    private final MessageId replyToId;
    private final String clientMessageId;
    private MessageStatus status;
    private Instant deletedAt;
    private Instant editedAt;
    private final Instant createdAt;
    private final List<DomainEvent> events = new ArrayList<>();

    public static Message create(MessageId id, RoomId roomId, UserId sender,
                                  long seq, MessageType type, String content,
                                  Map<String, Object> metadata, MessageId replyTo,
                                  String clientMessageId, Instant now) {
        return new Message(id, roomId, sender, seq, type,
            type == MessageType.TEXT ? sanitize(content) : content,
            metadata, replyTo, clientMessageId, MessageStatus.SENT, now);
    }

    public void edit(String newContent, Instant now) {
        require(status == MessageStatus.SENT, "cannot edit");
        require(type == MessageType.TEXT, "only text editable");
        require(Duration.between(createdAt, now).toMinutes() < 5, "edit window");
        this.content = sanitize(newContent);
        this.editedAt = now;
        events.add(new MessageEdited(id, roomId, content, now));
    }

    public void deleteByOwner(Instant now) {
        var withinFive = Duration.between(createdAt, now).toMinutes() < 5;
        this.status = withinFive ? MessageStatus.DELETED : MessageStatus.HIDDEN;
        this.deletedAt = now;
        events.add(withinFive
            ? new MessageHardDeleted(id, roomId, now)
            : new MessageSoftDeleted(id, roomId, now));
    }

    public void hideByAdmin(String reason, Instant now) {
        this.status = MessageStatus.HIDDEN;
        events.add(new MessageHidden(id, roomId, reason, now));
    }

    public boolean isVisible() {
        return status != MessageStatus.DELETED && status != MessageStatus.HIDDEN;
    }
}
```

---

## 불변식

- 5분 안 본인 삭제 = HARD DELETE.
- 5분 후 본인 삭제 = HIDDEN (soft).
- TEXT 만 편집 가능.
- seq 변경 X.

---

## 이벤트

- MessageSent
- MessageEdited
- MessageHardDeleted
- MessageSoftDeleted
- MessageHidden (admin)

---

## 관련

- [[domain-model|↑ hub]]
- [[../enums/message-type]] / [[../enums/message-status]]
- [[../database/messages-table]]
- [[../design-decisions/message-ordering]]
