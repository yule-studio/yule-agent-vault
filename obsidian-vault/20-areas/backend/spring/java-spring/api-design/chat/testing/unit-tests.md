---
title: "Unit tests"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:14:00+09:00
tags: [backend, java-spring, api-design, chat, testing, unit]
---

# Unit tests

**[[testing|↑ hub]]**

---

## Message aggregate

```java
@Test void edit_within_5min_ok() {
    var msg = textMessage(now);
    msg.edit("new content", now.plusSeconds(60));
    assertThat(msg.editedAt()).isNotNull();
}

@Test void edit_after_5min_throws() {
    var msg = textMessage(now);
    assertThatThrownBy(() -> msg.edit("new", now.plusMinutes(6)))
        .hasMessageContaining("edit window");
}

@Test void delete_within_5min_hard() {
    var msg = textMessage(now);
    msg.deleteByOwner(now.plusSeconds(60));
    assertThat(msg.status()).isEqualTo(MessageStatus.DELETED);
}

@Test void delete_after_5min_soft() {
    var msg = textMessage(now);
    msg.deleteByOwner(now.plusMinutes(10));
    assertThat(msg.status()).isEqualTo(MessageStatus.HIDDEN);
}
```

## Room aggregate

```java
@Test void direct_two_user() {
    var room = Room.createDirect(RoomId.next(), userA, userB, now);
    assertThat(room.type()).isEqualTo(RoomType.DIRECT);
    assertThat(room.maxMembers()).isEqualTo(2);
}

@Test void group_max_member_throws() {
    var room = Room.createGroup(RoomId.next(), owner, "Test", 2, now);
    room.addMember(userA);
    room.addMember(userB);
    assertThatThrownBy(() -> room.addMember(userC))
        .hasMessageContaining("max members");
}
```

## Sanitizer

```java
@Test void xss_sanitized() {
    var dirty = "Hello <script>alert(1)</script>";
    var clean = sanitizer.sanitize(dirty);
    assertThat(clean).doesNotContain("<script>");
}
```

---

## 관련

- [[testing|↑ hub]]
- [[../domain-model/message-aggregate]]
