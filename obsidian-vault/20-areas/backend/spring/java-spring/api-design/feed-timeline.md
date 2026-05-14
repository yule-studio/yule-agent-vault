---
title: "피드 / 타임라인 (네이버 카페·인스타 스타일)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - feed
  - timeline
---

# 피드 / 타임라인 (네이버 카페·인스타 스타일)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | fan-out on read vs write / cursor / 좋아요 |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]]. 검색 패턴: [[product-search#2]] (cursor pagination 동일).

---

## 1. 무엇을 만드는가

```
POST   /api/v1/posts                  # 글 작성
GET    /api/v1/posts/{id}             # 글 상세
PATCH  /api/v1/posts/{id}             # 수정
DELETE /api/v1/posts/{id}             # soft delete

POST   /api/v1/posts/{id}/like        # 좋아요 토글
POST   /api/v1/posts/{id}/bookmark    # 북마크
GET    /api/v1/posts/{id}/comments    # 댓글
POST   /api/v1/posts/{id}/comments    # 댓글 작성

GET    /api/v1/feed                   # 내 피드 (팔로우한 사람 / 추천)
GET    /api/v1/users/{id}/posts       # 특정 사용자의 글
GET    /api/v1/feed/trending          # 인기 / 트렌딩
```

### 1.1 도메인 모델

- **Post** — 글 (title, content, images, author, createdAt)
- **PostLike** — 좋아요 (user × post unique)
- **PostBookmark** — 북마크
- **PostComment** — 댓글 (nested 옵션)
- **Follow** — user → user

---

## 2. 피드 생성 전략 — Fan-out on Read vs Write

| | Fan-out on Read | Fan-out on Write | Hybrid |
| --- | --- | --- | --- |
| **방식** | 조회 시 follow한 user 의 posts 모두 JOIN | 글 작성 시 follower 의 feed 에 미리 push | 일반 user = read / VIP = write |
| **글 작성 비용** | O(1) | O(follower 수) |  |
| **피드 조회 비용** | O(N follow × M posts) | O(1) | 혼합 |
| **저장소** | DB JOIN | 추가 feed 테이블 / Redis | 둘 다 |
| **적합** | follow 수 적음 (< 1000) | follow 수 많음 (인스타) | Twitter / 인기 user |

**규모별 결정**:
- **소규모 / 카페식** (작은 그룹) → **Read** (단순)
- **인스타·페이스북식** (수십만 follow) → **Write** (push)
- **트위터식** (스타 + 일반) → **Hybrid**

본 레시피는 **Read 기본 + Hybrid 변형 §6**.

---

## 3. DB 스키마

```sql
CREATE TABLE posts (
    id          CHAR(26) PRIMARY KEY,
    author_id   CHAR(26) NOT NULL REFERENCES users(id),
    title       VARCHAR(200),
    content     TEXT NOT NULL,
    like_count  INTEGER NOT NULL DEFAULT 0,
    comment_count INTEGER NOT NULL DEFAULT 0,
    view_count  BIGINT NOT NULL DEFAULT 0,
    status      VARCHAR(20) NOT NULL DEFAULT 'PUBLISHED',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_posts_author_created ON posts (author_id, created_at DESC);
CREATE INDEX ix_posts_created ON posts (status, created_at DESC, id DESC) WHERE status = 'PUBLISHED';

CREATE TABLE post_images (
    id        CHAR(26) PRIMARY KEY,
    post_id   CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    image_url VARCHAR(500) NOT NULL,
    sort      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE post_likes (
    post_id    CHAR(26) NOT NULL REFERENCES posts(id),
    user_id    CHAR(26) NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (post_id, user_id)
);
CREATE INDEX ix_post_likes_user ON post_likes (user_id, created_at DESC);

CREATE TABLE post_bookmarks (
    post_id    CHAR(26) NOT NULL REFERENCES posts(id),
    user_id    CHAR(26) NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (post_id, user_id)
);

CREATE TABLE post_comments (
    id         CHAR(26) PRIMARY KEY,
    post_id    CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id  CHAR(26) NOT NULL REFERENCES users(id),
    parent_id  CHAR(26) REFERENCES post_comments(id),     -- 1-depth nesting (대댓글)
    content    TEXT NOT NULL,
    deleted    BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_post_comments_post ON post_comments (post_id, created_at);

CREATE TABLE follows (
    follower_id  CHAR(26) NOT NULL REFERENCES users(id),
    followee_id  CHAR(26) NOT NULL REFERENCES users(id),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX ix_follows_follower ON follows (follower_id);
CREATE INDEX ix_follows_followee ON follows (followee_id);
```

`like_count` / `comment_count` 는 **비정규화** — 매번 COUNT(*) 안 함. 변동 시 atomic UPDATE.

---

## 4. Feed 조회 (Fan-out on Read)

```sql
-- 내 피드 = 팔로우한 사람들의 글
SELECT p.* FROM posts p
WHERE p.status = 'PUBLISHED'
  AND p.author_id IN (
      SELECT followee_id FROM follows WHERE follower_id = :userId
      UNION
      SELECT :userId                               -- 자기 글도 포함
  )
  AND (p.created_at, p.id) < (:cursorCreatedAt, :cursorId)
ORDER BY p.created_at DESC, p.id DESC
LIMIT 20;
```

```java
@Query(value = """
    SELECT p.id, p.author_id AS authorId, p.title, p.content,
           p.like_count AS likeCount, p.comment_count AS commentCount,
           p.created_at AS createdAt
    FROM posts p
    WHERE p.status = 'PUBLISHED'
      AND p.author_id IN (
          SELECT followee_id FROM follows WHERE follower_id = :userId
          UNION SELECT CAST(:userId AS CHAR(26))
      )
      AND ((:cursorCreated IS NULL)
           OR (p.created_at, p.id) < (:cursorCreated, :cursorId))
    ORDER BY p.created_at DESC, p.id DESC
    LIMIT :limit
    """, nativeQuery = true)
List<FeedView> feed(@Param("userId") String userId,
                    @Param("cursorCreated") Instant cursorCreated,
                    @Param("cursorId") String cursorId,
                    @Param("limit") int limit);
```

> **함정**: `follower 수 × 평균 post 수` 가 큰 사용자는 느림. **§6 Hybrid** 로.

---

## 5. 좋아요 토글 — atomic UPDATE

```java
@Transactional
public boolean toggle(PostId postId, UserId userId) {
    var existed = postLikes.exists(postId, userId);
    if (existed) {
        postLikes.delete(postId, userId);
        postRepo.decrementLikeCount(postId);            // UPDATE posts SET like_count = like_count - 1 WHERE id = ?
    } else {
        try {
            postLikes.save(postId, userId, Instant.now(clock));
            postRepo.incrementLikeCount(postId);
        } catch (DataIntegrityViolationException e) {
            // 동시 클릭 — 이미 존재
        }
    }
    return !existed;
}
```

```java
// JPA modifying — 동시성 안전
@Modifying(clearAutomatically = true)
@Query("update PostJpaEntity p set p.likeCount = p.likeCount + 1 where p.id = :id")
int incrementLikeCount(@Param("id") String id);
```

→ row lock minimal. hot post (좋아요 폭주) 대비:
- Redis INCR 로 카운터 + 1분 batch DB sync
- 또는 LIKE atomic 자체는 빠르므로 일반엔 OK

---

## 6. Hybrid Fan-out (VIP user)

VIP user (follower > 10000) 의 글은 작성 시점에 follower 의 feed cache 에 push.

```sql
CREATE TABLE feed_cache (
    user_id    CHAR(26) NOT NULL,
    post_id    CHAR(26) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, post_id)
);
CREATE INDEX ix_feed_cache_user_created ON feed_cache (user_id, created_at DESC);
```

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
@Async
public void onPostCreated(PostCreated event) {
    var author = userRepo.findById(event.authorId()).orElseThrow();
    if (author.followerCount() > 10_000) {
        // Hybrid Push — Kafka 로 비동기
        kafkaProducer.send("feed-fanout", new FanoutMessage(event.postId(), event.authorId()));
    }
    // 일반 user 의 글은 read 시점에 직접 query (push X)
}

// Kafka consumer
@KafkaListener(topics = "feed-fanout")
public void consume(FanoutMessage msg) {
    var followers = followRepo.followerIdsOf(msg.authorId());
    feedCacheRepo.batchInsert(followers, msg.postId(), Instant.now());
}
```

Feed 조회:
```sql
-- VIP push + 일반 user 의 read query UNION
SELECT * FROM (
    SELECT post_id, created_at FROM feed_cache WHERE user_id = :userId
    UNION
    SELECT p.id, p.created_at FROM posts p
    WHERE p.author_id IN (
        SELECT followee_id FROM follows WHERE follower_id = :userId
        AND followee_id NOT IN (SELECT id FROM users WHERE is_vip = true)
    )
) combined
ORDER BY created_at DESC LIMIT 20;
```

> 복잡. **시작은 Fan-out on Read** + 성장 시 Hybrid.

---

## 7. 트렌딩 — 시간 가중 인기 글

```sql
-- 24시간 이내 글의 점수 = like_count + comment_count*2, decay 적용
SELECT id, title,
       (like_count + comment_count * 2)
       * exp(-extract(epoch from (now() - created_at)) / 86400.0)  -- hourly decay
       AS score
FROM posts
WHERE status = 'PUBLISHED'
  AND created_at > now() - INTERVAL '7 days'
ORDER BY score DESC
LIMIT 50;
```

→ Redis ZSET (`zincrby trending {postId} score`) + 매 분 decay 도 옵션.

---

## 8. Controller (핵심만)

```java
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class FeedController {

    private final FeedQueryService feed;
    private final TogglePostLikeUseCase toggleLike;

    @GetMapping("/feed")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<CommonResponse<FeedResponse>> myFeed(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") @Min(1) @Max(50) int limit,
        Authentication auth
    ) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            feed.myFeed(new UserId(auth.getName()), cursor, limit), "조회"));
    }

    @PostMapping("/posts/{postId}/like")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<CommonResponse<Map<String, Boolean>>> like(
        @PathVariable String postId, Authentication auth
    ) {
        boolean active = toggleLike.handle(new PostId(postId), new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("active", active), active ? "좋아요" : "취소"));
    }
}
```

---

## 9. 함정 모음

### 함정 1 — Feed 매번 N+1
follow → followee posts → 작성자 → 이미지 N+1. **JOIN + DTO projection**.

### 함정 2 — `like_count` 매번 COUNT
hot post = COUNT 폭증. **비정규화 column**.

### 함정 3 — Fan-out write 무지성
follower 100만 = 100만 row insert. **VIP 만 push, async, Kafka**.

### 함정 4 — 댓글 깊이 무제한
N-level nesting = 트리 쿼리 폭증. **1-level (post → 댓글 → 대댓글)** 권장.

### 함정 5 — 좋아요 race
A 가 like + B 가 like 동시 = like_count +2 만? UPDATE atomic increment.

### 함정 6 — 글 삭제 시 like / comment 같이 cascade
CASCADE = O(N) DELETE. **soft delete + lazy cleanup**.

### 함정 7 — 무한 스크롤 offset 사용
20페이지부터 느림. cursor (created_at, id) 페어.

### 함정 8 — 좋아요 알림 무제한
인기 글 = 알림 폭주. **digest 알림 (시간당 1회)**.

### 함정 9 — bookmark 따로 query
"내가 좋아한 글" + "내가 북마크한 글" 별도 API. UNION 또는 client-side merge.

### 함정 10 — 트렌딩 매번 SQL 계산
무거움. **5분 cache** 또는 Redis ZSET.

---

## 10. 운영 체크리스트

- [ ] like_count / comment_count 비정규화
- [ ] atomic UPDATE (race 안전)
- [ ] cursor pagination
- [ ] feed 의 follow JOIN 인덱스
- [ ] 트렌딩 캐시
- [ ] 댓글 1-level nesting
- [ ] VIP 식별 + push 분리 (Hybrid)
- [ ] 좋아요 / 댓글 알림 digest

---

## 11. 관련

- [[product-search]] — cursor pagination 동일
- [[chat-realtime]] — 알림 broadcast
- [[recommendation]] — 추천 feed 변형
- [[file-upload-s3]] — 글 이미지
- [[api-design|↑ api-design hub]]
