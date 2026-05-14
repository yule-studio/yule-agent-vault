---
title: "comments 테이블 — 댓글 / 대댓글 (2-level)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - comments-table
---

# comments 테이블 — 댓글 / 대댓글 (2-level)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 2-level 강제 (댓글 + 대댓글). 무한 depth X. 정책: [[../design-decisions/comment-structure]].

---

## 1. Schema

```sql
-- V8__create_comments.sql
CREATE TABLE comments (
    id          CHAR(26) PRIMARY KEY,
    post_id     CHAR(26) NOT NULL REFERENCES posts(id),
    parent_id   CHAR(26) REFERENCES comments(id),       -- NULL=댓글, 값=대댓글
    author_id   CHAR(26) NOT NULL,
    content     TEXT NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    like_count  INTEGER NOT NULL DEFAULT 0,
    version     BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ,

    CONSTRAINT chk_comments_status
        CHECK (status IN ('ACTIVE', 'HIDDEN', 'DELETED')),
    CONSTRAINT chk_comments_content_length
        CHECK (length(content) BETWEEN 1 AND 5000),
    CONSTRAINT chk_comments_like_positive
        CHECK (like_count >= 0)
);

-- 2-level 강제 (대대댓글 차단)
CREATE OR REPLACE FUNCTION enforce_comment_depth() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.parent_id IS NOT NULL THEN
        IF (SELECT parent_id FROM comments WHERE id = NEW.parent_id) IS NOT NULL THEN
            RAISE EXCEPTION 'comments depth max 2 (no grandchild)';
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_enforce_comment_depth
    BEFORE INSERT OR UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION enforce_comment_depth();
```

---

## 2. 인덱스

```sql
-- 댓글 list (post 별, parent 별)
CREATE INDEX ix_comments_post_parent_created ON comments
    (post_id, parent_id, created_at) WHERE status = 'ACTIVE';

-- 작성자별
CREATE INDEX ix_comments_author_created ON comments
    (author_id, created_at DESC) WHERE status = 'ACTIVE';
```

---

## 3. 컬럼 "왜"

### 3.1 `parent_id` (NULL=댓글, 값=대댓글)

- 단일 테이블에서 self-reference (adjacency list).
- NULL = depth 0 (댓글).
- 값 = depth 1 (대댓글, parent 의 자식).

### 3.2 왜 trigger (대대댓글 차단)

- application 만 의존 시 — 직접 SQL INSERT 우회 가능.
- DB trigger = 마지막 방어선.
- 자세히: [[../design-decisions/comment-structure#3.1]].

### 3.3 `like_count` 컬럼

- 댓글에도 좋아요 (post 와 동일 패턴).
- Redis + 1h batch sync.

### 3.4 soft delete (status=DELETED + content)

- 부모 댓글 삭제 시 자식 (대댓글) 보존.
- content="[삭제된 댓글입니다]" 마스킹.

---

## 4. 조회 — Tree 구성

```sql
-- 한 query 로 모든 댓글 + 대댓글
SELECT * FROM comments
WHERE post_id = ? AND status = 'ACTIVE'
ORDER BY COALESCE(parent_id, id), created_at;
```

```java
// application 단 tree
Map<String, List<Comment>> byParent = comments.stream()
    .filter(c -> c.parentId() != null)
    .collect(Collectors.groupingBy(Comment::parentId));

return comments.stream()
    .filter(c -> c.parentId() == null)
    .map(c -> new CommentTree(c, byParent.getOrDefault(c.id(), List.of())))
    .toList();
```

자세히: [[../design-decisions/comment-structure#4]].

---

## 5. JPA Entity

```java
@Entity @Table(name = "comments")
public class CommentJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "post_id", nullable = false, length = 26) private String postId;
    @Column(name = "parent_id", length = 26) private String parentId;
    @Column(name = "author_id", nullable = false, length = 26) private String authorId;
    @Column(nullable = false, columnDefinition = "TEXT") private String content;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private CommentStatus status;

    @Column(name = "like_count", nullable = false) private int likeCount;
    @Version @Column(nullable = false) private long version;
    @Column(name = "created_at", nullable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;
    @Column(name = "deleted_at") private Instant deletedAt;
}
```

---

## 6. 함정

### 함정 1 — Trigger 없이 application 만
직접 SQL 우회로 대대댓글 가능.
→ DB trigger.

### 함정 2 — Tree 구성 시 N+1
댓글마다 자식 조회.
→ 한 query + Map 으로 그룹화.

### 함정 3 — 부모 hard delete
자식 (대댓글) 고아 row.
→ soft delete (status=DELETED + content 마스킹).

### 함정 4 — `parent_id` FK 자체 참조 cascade
ON DELETE CASCADE 시 부모 삭제하면 자식 전부 삭제.
→ ON DELETE NO ACTION (또는 RESTRICT) + soft delete 강제.

### 함정 5 — comment_count race
post.comment_count 가 actual count 와 mismatch.
→ INSERT/DELETE 시 application 단 +1/-1 또는 1h batch 보정.

### 함정 6 — content 길이 무제한
abuse (10MB 댓글).
→ CHECK 1-5000.

### 함정 7 — 인덱스 (post_id, parent_id, created_at) 없음
댓글 tree query 풀스캔.
→ composite 인덱스.

---

## 7. 관련

- [[database|↑ hub]]
- [[../design-decisions/comment-structure]] — 2-level 정책
- [[posts-table]] — comment_count 컬럼
- [[likes-tables]] — comment_likes
