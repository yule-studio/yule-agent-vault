---
title: "Room CRUD 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:46:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, room]
---

# Room CRUD 구현

**[[implementation|↑ hub]]**

---

## 1. REST API

```http
POST   /api/v1/rooms                      (DIRECT 또는 GROUP)
GET    /api/v1/rooms                      (내 chat list)
GET    /api/v1/rooms/{id}                 (방 상세)
PATCH  /api/v1/rooms/{id}                 (이름 / 프로필)
DELETE /api/v1/rooms/{id}                 (방장만)
POST   /api/v1/rooms/{id}/members         (초대)
DELETE /api/v1/rooms/{id}/members/{userId} (강퇴 / 자진 퇴장)
PATCH  /api/v1/rooms/{id}/members/{userId}/role  (방장 양도)
GET    /api/v1/rooms/search?q=...         (OPEN 검색)
POST   /api/v1/rooms/{id}/join            (OPEN 가입)
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class RoomService {

    private final RoomRepository rooms;
    private final RoomMemberRepository members;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public RoomId createDirect(UserId me, UserId other) {
        // 이미 있으면 재사용 (UNIQUE pair)
        var existing = rooms.findDirectPair(me, other);
        if (existing.isPresent()) return existing.get().id();

        var room = Room.createDirect(RoomId.next(), me, other, clock.now());
        rooms.save(room);
        members.saveAll(List.of(
            RoomMember.of(room.id(), me, MemberRole.MEMBER, clock.now()),
            RoomMember.of(room.id(), other, MemberRole.MEMBER, clock.now())));
        return room.id();
    }

    @Transactional
    public RoomId createGroup(UserId owner, String name, List<UserId> initialMembers) {
        require(initialMembers.size() < 500);
        var room = Room.createGroup(RoomId.next(), owner, name, 500, clock.now());
        rooms.save(room);

        var memberList = new ArrayList<RoomMember>();
        memberList.add(RoomMember.of(room.id(), owner, MemberRole.OWNER, clock.now()));
        for (var u : initialMembers) {
            memberList.add(RoomMember.of(room.id(), u, MemberRole.MEMBER, clock.now()));
        }
        members.saveAll(memberList);
        return room.id();
    }

    @Transactional
    public void invite(RoomId roomId, UserId actor, UserId target) {
        var room = rooms.findById(roomId).orElseThrow();
        var actorMember = members.find(roomId, actor).orElseThrow();
        require(actorMember.role() != MemberRole.MEMBER || allowMemberInvite,
                "no permission");
        require(room.memberCount() < room.maxMembers(), "full");

        room.addMember(target);
        members.save(RoomMember.of(roomId, target, MemberRole.MEMBER, clock.now()));
        rooms.save(room);
    }

    @Transactional
    public void leave(RoomId roomId, UserId user) {
        var member = members.find(roomId, user).orElseThrow();
        member.markLeft(clock.now());
        members.save(member);

        var room = rooms.findById(roomId).orElseThrow();
        room.removeMember(user);
        rooms.save(room);
        // 방장 자동 양도 (남은 admin → member 순)
    }
}
```

---

## 3. 함정

- DIRECT 중복 생성 (race) → DB UNIQUE.
- max_members 검증 누락.
- 방장 탈퇴 시 자동 양도 X → 영구 잠김.

---

## 관련

- [[implementation|↑ hub]]
- [[../domain-model/room-aggregate]]
- [[../database/rooms-table]]
- [[../design-decisions/room-types]]
