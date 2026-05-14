---
title: "Room Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:52:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, aggregate, room]
---

# Room Aggregate

**[[domain-model|↑ hub]]**

---

```java
public final class Room {
    private final RoomId id;
    private final RoomType type;
    private String name;
    private String description;
    private int memberCount;
    private final int maxMembers;
    private final UserId creatorId;
    private boolean searchable;
    private String passwordHash;
    private boolean e2eeEnabled;
    private long lastSeq;
    private Instant lastMessageAt;
    private long version;
    private Instant deletedAt;

    public static Room createDirect(RoomId id, UserId a, UserId b, Instant now) {
        require(!a.equals(b), "self DM");
        return new Room(id, RoomType.DIRECT, null, null, 0, 2, null, false, null, false, now);
    }

    public static Room createGroup(RoomId id, UserId owner, String name, int max, Instant now) {
        return new Room(id, RoomType.GROUP, name, null, 0, max, owner, false, null, false, now);
    }

    public void addMember(UserId user) {
        require(memberCount < maxMembers, "max members reached");
        memberCount++;
    }

    public void removeMember(UserId user) {
        require(memberCount > 0);
        memberCount--;
    }

    public boolean canSendTo(UserId sender, BlockChecker blocks) {
        require(deletedAt == null, "room deleted");
        // members 검증은 RoomMember entity 에서
        return !blocks.isBlocked(sender);
    }

    public long incrementSeq() {
        return ++lastSeq;
    }
}
```

---

## 불변식

- DIRECT: maxMembers=2 고정.
- memberCount ≤ maxMembers.
- DIRECT type 변경 X (영구).
- creatorId == OWNER (GROUP 만).

---

## 관련

- [[domain-model|↑ hub]]
- [[../enums/room-type]]
- [[../database/rooms-table]]
