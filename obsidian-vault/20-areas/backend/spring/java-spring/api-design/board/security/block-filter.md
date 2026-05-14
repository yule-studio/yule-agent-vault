---
title: "차단 사용자 filter — 모든 query 적용"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:48:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - security
  - block
---

# 차단 사용자 filter — 모든 query 적용

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[security|↑ security hub]]**

> 양방향 block. 모든 list query 에 filter 적용. 정책: [[../design-decisions/block-policy]].

---

## 1. 적용 endpoint

모든 user-generated content list:
- `GET /boards/{id}/posts`
- `GET /posts/{id}/comments`
- `GET /posts/search`
- `GET /me/bookmarks` (북마크한 글의 작성자가 차단됐을 수도)
- `GET /tags/{tag}/posts`

→ filter 누락 시 차단한 user 의 글 봄 (의도 X).

---

## 2. 구현 — application 단 filter

```java
@Service
@RequiredArgsConstructor
public class BlockFilter {

    private final UserBlockRepository blocks;
    private final RedisTemplate<String, String> redis;

    public Set<UserId> getBlockedUsers(UserId viewerId) {
        if (viewerId == null) return Set.of();    // anonymous 는 차단 X

        String key = "user:blocks:" + viewerId.value();
        Set<String> cached = redis.opsForSet().members(key);
        if (cached != null && !cached.isEmpty()) {
            return cached.stream().map(UserId::new).collect(Collectors.toSet());
        }

        var fromDb = blocks.findRelatedTo(viewerId);
        redis.opsForSet().add(key, fromDb.stream().map(UserId::value).toArray(String[]::new));
        redis.expire(key, Duration.ofHours(1));
        return fromDb;
    }
}
```

```java
public List<Post> listPosts(BoardId boardId, AuthUser viewer, Cursor cursor) {
    var posts = postRepo.findByBoard(boardId, cursor, 21);    // limit+1
    var blocked = blockFilter.getBlockedUsers(viewer.id());
    return posts.stream()
        .filter(p -> !blocked.contains(p.authorId()))
        .limit(20)
        .toList();
}
```

### 2.1 왜 application filter (DB JOIN X)

- block list 가 보통 작음 (< 100).
- DB JOIN — block list 가 1000+ 시 부담.
- application 단 = block list cache 활용.

### 2.2 왜 limit+1 fetch

- filter 후 표시할 row 수 보장.
- 단점: filter 가 많으면 hasNext 정확도 ↓ — 추가 fetch 옵션.

---

## 3. ADMIN 의 filter bypass

```java
public List<Post> listPostsForAdmin(BoardId boardId, AuthUser admin, Cursor cursor) {
    // admin 은 차단 filter 적용 X (모더 위함)
    return postRepo.findByBoard(boardId, cursor, 20);
}
```

→ admin 은 모든 글 봐야 모더 가능.

---

## 4. DM / 멘션 차단 (옵션)

차단된 user 가 본인을 멘션하면?
- DM 시스템에선 차단 시 메시지 발송 X.
- 게시판에서 @멘션 = 알림 X.

```java
public void notifyMention(UserId mentionedId, PostId postId) {
    var mentioner = posts.findById(postId).get().authorId();
    if (blocks.isBlocked(mentionedId, mentioner)) return;     // skip
    notification.send(mentionedId, ...);
}
```

---

## 5. Cache 동기화

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onBlocked(UserBlocked event) {
    // 양쪽 user 의 cache invalidate
    redis.delete("user:blocks:" + event.blockerId());
    redis.delete("user:blocks:" + event.blockedId());
}
```

→ 차단 / 해제 시 즉시 cache 무효화.

---

## 6. 함정

### 함정 1 — Filter 누락
차단한 user 의 글 / 댓글 계속 봄.
→ 모든 list query 에.

### 함정 2 — Admin 도 filter 적용
admin 이 신고 글 모더 시 차단된 user 글 안 보임.
→ admin role bypass.

### 함정 3 — Cache 갱신 X
차단 후 옛 cache 의 글 봄.
→ 차단 / 해제 시 cache DEL.

### 함정 4 — block list 무한
DB JOIN 부담 ↑.
→ 1000 한도 + cache.

### 함정 5 — anonymous 의 filter
viewerId null 인데 filter 적용 → NullPointer.
→ anonymous 는 빈 Set.

### 함정 6 — cursor pagination 의 정확도 ↓
filter 후 page 가 short.
→ limit+1 또는 over-fetch.

---

## 7. 관련

- [[security|↑ hub]]
- [[../design-decisions/block-policy]] — 정책
- [[../database/user-blocks-table]]
