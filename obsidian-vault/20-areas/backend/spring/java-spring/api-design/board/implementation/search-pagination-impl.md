---
title: "검색 / 정렬 / 페이지네이션 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T14:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - implementation
  - search
---

# 검색 / 정렬 / 페이지네이션 구현

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ implementation hub]]**

> 검색 (DB ILIKE / FTS) + 정렬 (latest / hot / top) + cursor pagination.

---

## 1. API spec

```http
# 게시판 내 목록 (정렬)
GET /api/v1/boards/{boardId}/posts?sort=hot&cursor=...&limit=20

# 검색
GET /api/v1/posts/search?q=맛집&boardId=01HZ...&sort=relevance

# 태그
GET /api/v1/posts?tag=강남&sort=latest
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class PostSearchService {

    private final PostRepository posts;
    private final BlockFilter blockFilter;

    @Transactional(readOnly = true)
    public PageResponse<PostResponse> listByBoard(
        BoardId boardId, PostSort sort, Cursor cursor, int limit, AuthUser viewer
    ) {
        // limit+1 fetch (hasNext 판단 + block filter buffer)
        var posts = switch (sort) {
            case LATEST -> postRepo.findByBoardOrderByCreated(boardId, cursor, limit + 1);
            case HOT -> postRepo.findByBoardOrderByHotScore(boardId, cursor, limit + 1);
            case TOP -> postRepo.findByBoardOrderByLikeCount(boardId, cursor, limit + 1);
        };

        // block filter
        var blocked = blockFilter.getBlockedUsers(viewer.id());
        var filtered = posts.stream()
            .filter(p -> !blocked.contains(p.authorId()))
            .limit(limit)
            .toList();

        boolean hasNext = posts.size() > limit;
        String nextCursor = hasNext ? Cursor.of(filtered.get(filtered.size() - 1), sort).encode() : null;

        return new PageResponse<>(
            filtered.stream().map(p -> PostResponse.of(p, viewer)).toList(),
            nextCursor, hasNext);
    }

    @Transactional(readOnly = true)
    public PageResponse<PostResponse> search(
        String q, BoardId boardId, Cursor cursor, int limit, AuthUser viewer
    ) {
        if (q == null || q.isBlank())
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT, "query required");
        if (q.length() > 100)
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT, "query too long");

        var posts = postRepo.search(q, boardId, cursor, limit + 1);
        var blocked = blockFilter.getBlockedUsers(viewer.id());
        // filter + limit + cursor
        // ...
    }
}
```

### 2.1 왜 `limit + 1`

- 21 row fetch → 20 표시 + hasNext = true 또는 nothing.
- 추가 query 없이 hasNext 판단.

### 2.2 왜 block filter 가 limit 후 (보정 X)

- block list 보통 작음 (< 100).
- filter 후 row 수 부족 시 — fetch more 필요 (over-fetch 옵션).
- 본 vault: 단순화 — 가끔 18-19 row 응답.

---

## 3. Cursor 구현

```java
public record Cursor(double primarySort, String id, PostSort type) {
    public String encode() {
        var json = Map.of(
            "primary", primarySort,
            "id", id,
            "type", type.name()
        );
        return Base64.getUrlEncoder().encodeToString(
            objectMapper.writeValueAsBytes(json));
    }

    public static Cursor decode(String encoded, PostSort sort) {
        if (encoded == null) return first(sort);
        var bytes = Base64.getUrlDecoder().decode(encoded);
        var json = objectMapper.readValue(bytes, Map.class);
        var type = PostSort.valueOf((String) json.get("type"));
        if (type != sort)
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT, "cursor type mismatch");
        return new Cursor(
            ((Number) json.get("primary")).doubleValue(),
            (String) json.get("id"),
            type);
    }

    public static Cursor first(PostSort sort) {
        return switch (sort) {
            case LATEST -> new Cursor(Double.MAX_VALUE, "Z", sort);
            case HOT -> new Cursor(Double.MAX_VALUE, "Z", sort);
            case TOP -> new Cursor(Double.MAX_VALUE, "Z", sort);
        };
    }

    public static Cursor of(Post post, PostSort sort) {
        return switch (sort) {
            case LATEST -> new Cursor(post.createdAt().getEpochSecond(), post.id().value(), sort);
            case HOT -> new Cursor(post.hotScore(), post.id().value(), sort);
            case TOP -> new Cursor(post.likeCount(), post.id().value(), sort);
        };
    }
}
```

---

## 4. Repository — JPA Adapter

```java
@Query("""
    SELECT p FROM PostJpaEntity p
    WHERE p.boardId = :boardId
      AND p.status = 'PUBLISHED'
      AND (p.hotScore < :hotScore
           OR (p.hotScore = :hotScore AND p.id < :id))
    ORDER BY p.hotScore DESC, p.id DESC
""")
List<PostJpaEntity> findByBoardOrderByHotScore(
    @Param("boardId") String boardId,
    @Param("hotScore") double hotScore,
    @Param("id") String id,
    Pageable pageable);
```

→ tuple comparison + composite index 활용.

자세히: [[../design-decisions/pagination-strategy]].

---

## 5. FTS (PostgreSQL)

```sql
-- posts.content_tsv 가 generated (database/posts-table.md)
-- 검색
SELECT p.* FROM posts p
WHERE p.content_tsv @@ plainto_tsquery('korean', :query)
  AND p.status = 'PUBLISHED'
  AND p.board_id = :boardId
ORDER BY
  ts_rank(p.content_tsv, plainto_tsquery('korean', :query)) DESC,
  p.id DESC
LIMIT :limit OFFSET :offset;
```

→ GIN 인덱스 활용.

자세히: [[../design-decisions/search-strategy]].

---

## 6. Controller

```java
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class PostListController {

    private final PostSearchService service;

    @GetMapping("/boards/{boardId}/posts")
    public CommonResponse<PageResponse<PostResponse>> list(
        @PathVariable BoardId boardId,
        @RequestParam(defaultValue = "hot") PostSort sort,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") @Max(50) int limit,
        @AuthenticationPrincipal AuthUser auth
    ) {
        var parsedCursor = Cursor.decode(cursor, sort);
        return CommonResponse.success(ResponseCode.OK,
            service.listByBoard(boardId, sort, parsedCursor, limit, auth));
    }

    @GetMapping("/posts/search")
    public CommonResponse<PageResponse<PostResponse>> search(
        @RequestParam @Size(min = 1, max = 100) String q,
        @RequestParam(required = false) BoardId boardId,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") @Max(50) int limit,
        @AuthenticationPrincipal AuthUser auth
    ) {
        return CommonResponse.success(ResponseCode.OK,
            service.search(q, boardId, Cursor.decode(cursor, PostSort.RELEVANCE), limit, auth));
    }
}
```

---

## 7. 함정

### 함정 1 — Offset 깊은 page
page 100 = OFFSET 2000.
→ cursor.

### 함정 2 — Cursor type 변경 시 깨짐
sort=latest cursor 를 sort=hot 에 사용.
→ cursor 안 type 검증.

### 함정 3 — Limit 무제한
abuse.
→ max 50.

### 함정 4 — Block filter 누락
차단 user 글 노출.
→ 모든 list에 filter.

### 함정 5 — HIDDEN / DELETED 검색 결과 포함
모더된 글 노출.
→ WHERE status = 'PUBLISHED'.

### 함정 6 — 검색 query length 무제한
1MB query.
→ Bean Validation Size 100.

### 함정 7 — FTS dictionary 없이
영어 dictionary 로 한국어.
→ pgroonga / mecab-ko.

### 함정 8 — Index 없이 정렬
풀스캔.
→ (board_id, hot_score DESC, id DESC) composite.

---

## 8. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/search-strategy]] · [[../design-decisions/pagination-strategy]] · [[../design-decisions/hot-ranking]]
- [[../database/posts-table]] — content_tsv
- [[../security/block-filter]]
