---
title: "Search 구현 (PostgreSQL FTS)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:00:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, search]
---

# Search 구현 (PostgreSQL FTS)

**[[implementation|↑ hub]]**

---

## API

```http
GET /api/v1/rooms/{id}/search?q=회의&limit=20
GET /api/v1/search?q=회의&limit=20         # 모든 room (가입한)
```

---

## Service

```java
@Service
@RequiredArgsConstructor
public class SearchService {

    private final MessageQueryDao dao;
    private final RoomMemberRepository members;

    public List<SearchHit> search(UserId user, String q, RoomId roomScope, int limit) {
        // 사용자 가입 room 만
        var rooms = roomScope != null
            ? Set.of(roomScope)
            : members.roomsForUser(user);

        return dao.searchByFts(rooms, q, limit);
    }
}
```

```java
// MyBatis 또는 JPA native query
@Query(nativeQuery = true, value = """
    SELECT m.id, m.room_id, m.sender_id, m.content,
           ts_headline('korean', m.content, query) AS snippet,
           m.created_at
    FROM messages m, plainto_tsquery('korean', :q) query
    WHERE m.room_id IN (:rooms)
      AND m.content_tsv @@ query
      AND m.status = 'SENT'
      AND m.created_at > now() - INTERVAL '6 months'
    ORDER BY ts_rank(m.content_tsv, query) DESC, m.created_at DESC
    LIMIT :limit
""")
List<SearchHit> searchByFts(Collection<UUID> rooms, String q, int limit);
```

---

## 함정

- 모든 partition scan → `created_at` range.
- 가입 안 한 room 검색 → privacy 누설.
- 한국어 tokenizer 미설정.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/search-strategy]]
