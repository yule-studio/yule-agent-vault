---
title: "카테고리 / 태그 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T14:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - implementation
  - taxonomy
---

# 카테고리 / 태그 구현

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ implementation hub]]**

> Category (board 안 분류) + Tag (자유 입력 / 자동 생성).

---

## 1. Service — Tag 자동 생성

```java
@Service
@RequiredArgsConstructor
public class TagService {

    private final TagRepository tags;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public Set<TagId> resolveOrCreate(Set<String> rawNames) {
        Set<TagId> result = new HashSet<>();
        for (String raw : rawNames) {
            var normalized = normalize(raw);
            if (normalized.isEmpty()) continue;

            var existing = tags.findByName(normalized);
            if (existing.isPresent()) {
                result.add(existing.get().id());
                tags.incrementUsage(existing.get().id());
            } else {
                var newTag = new Tag(
                    new TagId(ids.next()),
                    normalized,
                    1,
                    Instant.now(clock)
                );
                try {
                    tags.save(newTag);
                    result.add(newTag.id());
                } catch (DataIntegrityViolationException e) {
                    // race — 동시 같은 이름 INSERT
                    result.add(tags.findByName(normalized).orElseThrow().id());
                }
            }
        }
        return result;
    }

    private String normalize(String raw) {
        return raw.trim().toLowerCase()
            .replaceAll("[\\s]+", "")            // 공백 제거
            .replaceAll("[^\\p{L}\\p{N}_-]", ""); // 특수문자 제외
    }

    public List<Tag> findPopular(int limit) {
        return tags.findAll(Sort.by("usageCount").descending(), limit);
    }

    public List<Tag> autocomplete(String prefix, int limit) {
        return tags.findByNameStartingWith(prefix, limit);     // trigram index 활용
    }
}
```

### 1.1 왜 normalize

- "맛집" / " 맛집 " / "맛집!" → 같은 태그.
- 일관성 + 인기 태그 정확도.

### 1.2 왜 race 시 retry

- 동시 다른 user 가 같은 새 tag 시도.
- DB UNIQUE 위반 → findByName 으로 fetch.

---

## 2. Service — Category

```java
@Service
@RequiredArgsConstructor
public class CategoryService {

    private final CategoryRepository categories;

    @Transactional(readOnly = true)
    public List<Category> findByBoard(BoardId boardId) {
        return categories.findActiveByBoardId(boardId);
    }

    public void validateBelongsToBoard(BoardId boardId, CategoryId categoryId) {
        var category = categories.findById(categoryId).orElseThrow();
        if (!category.boardId().equals(boardId))
            throw new BusinessException(ResponseCode.INVALID_INPUT_FORMAT,
                "category not belongs to this board");
    }
}
```

→ admin 만 category CRUD (별도 endpoint).

---

## 3. Post 의 tag / category 매핑

```java
// PostService.create() 안에서
postCategoryRepo.save(post.id(), categoryId);
var tagIds = tagService.resolveOrCreate(cmd.tags());
postTagRepo.saveAll(post.id(), tagIds);
```

---

## 4. 조회 — tag 별 posts

```sql
SELECT p.* FROM posts p
JOIN post_tags pt ON pt.post_id = p.id
JOIN tags t ON t.id = pt.tag_id
WHERE t.name = :tagName
  AND p.status = 'PUBLISHED'
ORDER BY p.created_at DESC
LIMIT 20;
```

---

## 5. Controller

```java
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class TaxonomyController {

    @GetMapping("/boards/{boardId}/categories")
    public CommonResponse<List<CategoryResponse>> categories(@PathVariable BoardId boardId) {
        return CommonResponse.success(ResponseCode.OK,
            categoryService.findByBoard(boardId).stream().map(CategoryResponse::of).toList());
    }

    @GetMapping("/tags/popular")
    public CommonResponse<List<TagResponse>> popular(@RequestParam(defaultValue = "20") @Max(50) int limit) {
        return CommonResponse.success(ResponseCode.OK,
            tagService.findPopular(limit).stream().map(TagResponse::of).toList());
    }

    @GetMapping("/tags/autocomplete")
    public CommonResponse<List<TagResponse>> autocomplete(
        @RequestParam @Size(min = 1, max = 50) String prefix,
        @RequestParam(defaultValue = "10") @Max(20) int limit
    ) {
        return CommonResponse.success(ResponseCode.OK,
            tagService.autocomplete(prefix, limit).stream().map(TagResponse::of).toList());
    }
}
```

---

## 6. 함정

### 함정 1 — Tag normalize 없음
"맛집" / "맛집 " 다른 row.
→ trim + lowercase.

### 함정 2 — 동시 같은 새 tag race
DataIntegrityViolation.
→ try-catch + findByName fallback.

### 함정 3 — Tag 무한 생성 (abuse)
10만 tag.
→ post 당 max 10 + 사용자 별 rate limit.

### 함정 4 — Category 가 다른 board 의 것
post 의 board_id 와 category.board_id mismatch.
→ validateBelongsToBoard().

### 함정 5 — usage_count 매 INSERT
update 폭주.
→ 1h batch.

### 함정 6 — Autocomplete 풀스캔
trigram index 없음.
→ pg_trgm extension + GIN.

---

## 7. 관련

- [[implementation|↑ hub]]
- [[../database/categories-tags-tables]]
- [[post-crud-impl]] — Post 가 tag / category 사용
