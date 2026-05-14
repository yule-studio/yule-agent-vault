---
title: "Unit Tests — board"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - testing
  - unit-test
---

# Unit Tests — board

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ testing hub]]**

> 도메인 / VO / Service 단위 테스트. signup 의 패턴 + board 특화.

---

## 1. Post Aggregate

```java
class PostTest {

    @Test
    void create_raises_PostCreated_event() {
        var post = Post.create(
            new PostId(ulidString()),
            new BoardId(ulidString()),
            new UserId(ulidString()),
            "title", "content",
            PostVisibility.PUBLIC,
            Instant.parse("2026-05-15T00:00:00Z")
        );

        assertThat(post.status()).isEqualTo(PostStatus.PUBLISHED);
        var events = post.pullDomainEvents();
        assertThat(events).hasSize(1).first().isInstanceOf(PostCreated.class);
    }

    @Test
    void hide_only_when_PUBLISHED() {
        var post = newPublishedPost();
        post.hide("reason", now);
        assertThat(post.status()).isEqualTo(PostStatus.HIDDEN);

        assertThatThrownBy(() -> post.hide("again", now))
            .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void deleted_cannot_update() {
        var post = newPublishedPost();
        post.delete(now);
        assertThatThrownBy(() -> post.update("new", "new", now))
            .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void counter_negative_protection() {
        var post = newPublishedPost();
        post.decrementLike();
        post.decrementLike();
        assertThat(post.likeCount()).isEqualTo(0);   // not -2
    }
}
```

## 2. Comment Aggregate

```java
class CommentTest {

    @Test
    void reply_flattens_to_2_level() {
        var c1 = Comment.create(new CommentId(ulidString()), postId, authorId, "c1", now);
        var c2 = Comment.reply(new CommentId(ulidString()), c1, otherAuthor, "c2", now);

        // c2 가 c1 의 reply (parent_id = c1)
        assertThat(c2.parentId()).isEqualTo(c1.id());
        assertThat(c2.isReply()).isTrue();

        // c3 가 c2 에 대한 reply — 평탄화 (c1 의 reply 로)
        var c3 = Comment.reply(new CommentId(ulidString()), c2, anotherAuthor, "c3", now);
        assertThat(c3.parentId()).isEqualTo(c1.id());     // c2 가 아닌 c1
    }

    @Test
    void delete_with_children_masks_content() {
        var c1 = newComment();
        c1.delete(now);
        assertThat(c1.status()).isEqualTo(CommentStatus.DELETED);
        assertThat(c1.content()).contains("삭제");
    }
}
```

## 3. PostService — Mockito

```java
@ExtendWith(MockitoExtension.class)
class PostServiceTest {

    @Mock PostRepository posts;
    @Mock TagRepository tags;
    @Mock S3Client s3;
    @InjectMocks PostService service;

    @Test
    void create_validates_attachment_exists() {
        when(s3.exists("fake-key")).thenReturn(false);

        assertThatThrownBy(() -> service.create(
            cmdWithAttachment("fake-key"), authorId
        )).isInstanceOf(BusinessException.class)
          .hasMessageContaining("attachment not uploaded");
    }

    @Test
    void update_only_by_author() {
        var post = newPost(authorId);
        when(posts.findById(post.id())).thenReturn(Optional.of(post));

        assertThatThrownBy(() -> service.update(post.id(), updateCmd, otherUserId))
            .isInstanceOf(ForbiddenException.class);
    }
}
```

## 4. VO

```java
class PostIdTest {
    @Test
    void invalid_length_rejected() {
        assertThatThrownBy(() -> new PostId("too-short"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}

class CursorTest {
    @Test
    void encode_decode_roundtrip() {
        var c = new Cursor(123.45, "01HQ...", PostSort.HOT);
        var encoded = c.encode();
        var decoded = Cursor.decode(encoded, PostSort.HOT);
        assertThat(decoded).isEqualTo(c);
    }

    @Test
    void type_mismatch_throws() {
        var c = new Cursor(123.45, "01HQ...", PostSort.HOT);
        assertThatThrownBy(() -> Cursor.decode(c.encode(), PostSort.LATEST))
            .isInstanceOf(BusinessException.class);
    }
}
```

## 5. MarkdownRenderer

```java
class MarkdownRendererTest {
    @Test
    void sanitizes_script_tag() {
        var output = renderer.render("# Hello\n<script>alert(1)</script>");
        assertThat(output).contains("<h1>Hello</h1>");
        assertThat(output).doesNotContain("<script>");
    }

    @Test
    void rejects_javascript_url() {
        var output = renderer.render("[click](javascript:alert(1))");
        assertThat(output).doesNotContain("javascript:");
    }
}
```

---

## 6. 함정

자세히: [[../../signup/testing/unit-tests#7 함정]].

---

## 7. 관련

- [[testing|↑ hub]]
- [[../../signup/testing/unit-tests|↗ signup unit-tests]]
- [[../domain-model/post-aggregate]] · [[../domain-model/comment-aggregate]]
