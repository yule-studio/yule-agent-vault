---
title: "Comment Aggregate — 댓글 / 대댓글"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - domain-model
  - comment
---

# Comment Aggregate — 댓글 / 대댓글

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ domain-model hub]]**

> 2-level 강제 댓글. parent → child 만 (대대댓글 X).

---

## 1. 코드

```java
public final class Comment {

    private final CommentId id;
    private final PostId postId;
    private final CommentId parentId;       // NULL = 댓글 (depth 0)
    private final UserId authorId;
    private String content;
    private CommentStatus status;
    private int likeCount;
    private final Instant createdAt;
    private Instant updatedAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private Comment(CommentId id, PostId postId, CommentId parentId, UserId authorId,
                    String content, CommentStatus status, Instant createdAt) {
        this.id = id; this.postId = postId; this.parentId = parentId;
        this.authorId = authorId; this.content = content;
        this.status = status; this.createdAt = createdAt; this.updatedAt = createdAt;
    }

    /** 새 댓글 (parent_id=NULL) */
    public static Comment create(CommentId id, PostId postId, UserId authorId,
                                  String content, Instant now) {
        validateContent(content);
        var c = new Comment(id, postId, null, authorId, content,
                             CommentStatus.ACTIVE, now);
        c.events.add(new CommentCreated(id, postId, authorId, now));
        return c;
    }

    /** 대댓글 — parent 가 댓글이어야 (대대댓글 차단). */
    public static Comment reply(CommentId id, Comment parent, UserId authorId,
                                 String content, Instant now) {
        validateContent(content);
        // 2-level 강제
        var resolvedParent = parent.parentId != null ? parent.parentId : parent.id;

        var c = new Comment(id, parent.postId, resolvedParent, authorId, content,
                             CommentStatus.ACTIVE, now);
        c.events.add(new CommentReplied(id, parent.id, parent.postId, authorId, now));
        return c;
    }

    public void update(String newContent, Instant now) {
        if (status != CommentStatus.ACTIVE)
            throw new IllegalStateException("not active: " + status);
        validateContent(newContent);
        this.content = newContent;
        this.updatedAt = now;
    }

    public void hide(String reason, Instant now) {
        if (status != CommentStatus.ACTIVE) return;
        this.status = CommentStatus.HIDDEN;
        this.updatedAt = now;
        events.add(new CommentHidden(id, reason, now));
    }

    public void delete(Instant now) {
        if (status == CommentStatus.DELETED) return;
        this.status = CommentStatus.DELETED;
        this.content = "[삭제된 댓글입니다]";        // 자식 보존 위해 마스킹
        this.updatedAt = now;
        events.add(new CommentDeleted(id, now));
    }

    public void incrementLike() { this.likeCount++; }
    public void decrementLike() { this.likeCount = Math.max(0, this.likeCount - 1); }

    public boolean isOwnedBy(UserId userId) { return authorId.equals(userId); }
    public boolean isReply() { return parentId != null; }

    private static void validateContent(String content) {
        if (content == null || content.isBlank())
            throw new IllegalArgumentException("content required");
        if (content.length() > 5000)
            throw new IllegalArgumentException("content too long (max 5000)");
    }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events); events.clear(); return copy;
    }

    public static Comment reconstitute(CommentId id, PostId postId, CommentId parentId,
                                        UserId authorId, String content,
                                        CommentStatus status, int likeCount,
                                        Instant createdAt, Instant updatedAt) {
        var c = new Comment(id, postId, parentId, authorId, content, status, createdAt);
        c.likeCount = likeCount;
        c.updatedAt = updatedAt;
        return c;
    }
    // accessors
}
```

---

## 2. 주요 "왜"

### 2.1 왜 `reply()` 가 평탄화 (resolvedParent)

```java
var resolvedParent = parent.parentId != null ? parent.parentId : parent.id;
```

**케이스**:
1. user A 의 댓글 (parent_id=NULL).
2. user B 의 A 댓글에 대댓글 (parent_id=A).
3. user C 가 B 대댓글에 답글 시도.

→ C 의 parent_id = B 가 아닌 A (조부모). 평탄화.

**왜**
- 2-level 강제 — 대대댓글 차단.
- DB trigger 도 같은 효과 (이중 방어).

자세히: [[../design-decisions/comment-structure]].

### 2.2 왜 `delete()` 후 content 마스킹

- 자식 (대댓글) 보존 위해 row 자체 보존.
- content = "[삭제된 댓글입니다]" → user 가 본 내용은 숨김.

### 2.3 왜 `update()` 가 DELETED 시 reject

- 삭제된 댓글 복원해서 내용 변경 — 의도 불명.
- 안 하면: race (다른 곳에서 delete 처리 중 update) 시 모호.

### 2.4 왜 `reply()` 가 static factory (parent Comment 받음)

- parent 의 상태 (parentId) 검증 시점에 보장.
- application 단의 평탄화 로직 안 분산.

---

## 3. Domain Events

```java
public sealed interface DomainEvent permits
    CommentCreated, CommentReplied, CommentHidden, CommentDeleted { }

public record CommentCreated(CommentId id, PostId postId, UserId authorId, Instant occurredAt) implements DomainEvent {}
public record CommentReplied(CommentId id, CommentId parentId, PostId postId, UserId authorId, Instant occurredAt) implements DomainEvent {}
public record CommentHidden(CommentId id, String reason, Instant occurredAt) implements DomainEvent {}
public record CommentDeleted(CommentId id, Instant occurredAt) implements DomainEvent {}
```

**왜 `CommentReplied` 별도** (vs Created)

- Listener 가 분기 — 대댓글 알림 (부모 댓글 작성자에게) vs 댓글 알림 (post 작성자에게).
- 도메인 이벤트 의미 명확.

---

## 4. Application — Service 의 사용

```java
@Transactional
public Comment createComment(CommentCreateCmd cmd) {
    var post = posts.findById(cmd.postId())
        .orElseThrow(() -> new NotFoundException("post"));
    if (!post.isVisible())
        throw new ForbiddenException("post not visible");

    var commentId = new CommentId(idGenerator.next());

    Comment comment;
    if (cmd.parentId() != null) {
        var parent = comments.findById(cmd.parentId())
            .orElseThrow(() -> new NotFoundException("parent comment"));
        comment = Comment.reply(commentId, parent, cmd.authorId(), cmd.content(),
                                 Instant.now(clock));
    } else {
        comment = Comment.create(commentId, cmd.postId(), cmd.authorId(),
                                  cmd.content(), Instant.now(clock));
    }

    comments.save(comment);
    post.incrementComment();
    posts.save(post);

    comment.pullDomainEvents().forEach(eventPublisher::publishEvent);
    return comment;
}
```

---

## 5. 함정

### 함정 1 — `reply()` 평탄화 안 함
대대댓글 가능.
→ resolvedParent 적용.

### 함정 2 — `delete()` 가 hard delete
자식 (대댓글) 고아.
→ soft delete + content 마스킹.

### 함정 3 — DELETED 상태에서 update 허용
의도 불명 + 동시성 모호.
→ status 검증.

### 함정 4 — `decrement` 음수
counter < 0 가능.
→ Math.max(0, ...).

### 함정 5 — `content` 검증 누락
1MB 댓글 가능.
→ length 5000 max.

### 함정 6 — `parent` 가 다른 post 의 댓글
post A 댓글에 post B 의 reply 시도.
→ reply() 의 postId = parent.postId 강제.

### 함정 7 — `reconstitute()` 가 event 발행
DB load 마다 CommentCreated → 사용자 알림 무한.
→ events 빈 list.

### 함정 8 — Comment 가 Post 의 child (Aggregate 안)
N+1 / 메모리 부담.
→ 별도 Aggregate + PostId 참조.

자세히: [[aggregate-boundaries]].

---

## 6. 관련

- [[domain-model|↑ hub]]
- [[../database/comments-table]] — 매핑
- [[../design-decisions/comment-structure]] — 2-level 정책
- [[post-aggregate]] — Post 의 comment_count
- [[domain-events]]
