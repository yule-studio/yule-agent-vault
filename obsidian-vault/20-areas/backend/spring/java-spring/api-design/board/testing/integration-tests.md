---
title: "Integration Tests — board"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - testing
  - integration
---

# Integration Tests — board

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ testing hub]]**

> Testcontainers + counter race + AFTER_COMMIT + cursor pagination.

---

## 1. 환경

- PostgreSQL 16 (Testcontainers).
- Redis 7 (Testcontainers).
- LocalStack (S3 mock).
- WireMock (FCM).

자세히: [[../../signup/testing/integration-tests|↗ signup integration-tests]].

---

## 2. 핵심 시나리오

### 2.1 Counter race

```java
@Test
void concurrent_likes_only_one_row_per_user() throws Exception {
    var post = createPost();
    var executor = Executors.newFixedThreadPool(10);
    var latch = new CountDownLatch(1);
    var successCount = new AtomicInteger();

    var futures = IntStream.range(0, 10)
        .mapToObj(i -> CompletableFuture.runAsync(() -> {
            try {
                latch.await();
                likeService.likePost(userId, post.id());      // 같은 user 동시 10번
                successCount.incrementAndGet();
            } catch (Exception e) { /* race tolerance */ }
        }, executor))
        .toList();

    latch.countDown();
    futures.forEach(CompletableFuture::join);

    // 같은 user 의 좋아요 = 1 row (DB UNIQUE)
    assertThat(postLikeRepo.countByPostId(post.id())).isEqualTo(1);
    // counter sync 후
    triggerBatchSync();
    assertThat(postRepo.findById(post.id()).get().likeCount()).isEqualTo(1);
}
```

### 2.2 Cursor pagination — 새 글 추가

```java
@Test
void cursor_pagination_no_duplicates_on_new_posts() {
    // Given: 30 posts
    IntStream.range(0, 30).forEach(i -> createPost("post-" + i));

    // page 1 fetch
    var page1 = controller.list(boardId, PostSort.LATEST, null, 20);

    // 새 글 1개 추가
    createPost("new-post");

    // page 2 fetch (cursor 사용)
    var page2 = controller.list(boardId, PostSort.LATEST, page1.nextCursor(), 20);

    // 중복 X
    var allIds = Stream.concat(page1.data().stream(), page2.data().stream())
        .map(PostResponse::id).distinct().count();
    assertThat(allIds).isEqualTo(page1.data().size() + page2.data().size());
}
```

### 2.3 AFTER_COMMIT — outbox

```java
@Test
void post_create_enqueues_notifications_after_commit() throws InterruptedException {
    postService.create(cmd, authorId);

    // commit 후 listener — Awaitility
    await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
        assertThat(outboxRepo.findByUserId(followerUserId)).isNotEmpty();
    });
}

@Test
void rollback_does_not_enqueue() {
    assertThatThrownBy(() -> postService.create(invalidCmd, authorId))
        .isInstanceOf(BusinessException.class);
    assertThat(outboxRepo.findAll()).isEmpty();
}
```

### 2.4 차단 filter

```java
@Test
void blocked_user_posts_not_in_list() {
    var alice = createUser();
    var bob = createUser();

    createPost(bob);     // bob 글
    blockService.block(alice.id(), bob.id());

    var posts = postSearchService.listByBoard(boardId, PostSort.LATEST, null, 20, alice);

    assertThat(posts.data()).noneMatch(p -> p.authorId().equals(bob.id()));
}
```

### 2.5 자동 hide (5회 신고)

```java
@Test
void auto_hide_after_5_reports() {
    var post = createPost();

    for (int i = 0; i < 5; i++) {
        var reporter = createUser();
        reportService.submit(reporter.id(), TargetId.of(post.id()),
                              ReportReason.SPAM, null);
    }

    await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
        var fetched = postRepo.findById(post.id()).get();
        assertThat(fetched.status()).isEqualTo(PostStatus.HIDDEN);
    });

    // 작성자 알림 outbox
    assertThat(outbox.findByUserId(post.authorId())).isNotEmpty();
}
```

### 2.6 댓글 N+1 회피

```java
@Test
void comment_tree_no_n_plus_one() {
    var post = createPost();
    IntStream.range(0, 20).forEach(i -> {
        var c = createComment(post.id(), null);
        IntStream.range(0, 3).forEach(j -> createComment(post.id(), c.id()));
    });

    var queryCount = new AtomicInteger();
    statementInspector.setListener(stmt -> queryCount.incrementAndGet());

    var tree = commentService.findTree(post.id());

    assertThat(tree).hasSize(20);
    assertThat(tree.stream().mapToLong(t -> t.replies().size()).sum()).isEqualTo(60);
    assertThat(queryCount.get()).isLessThanOrEqualTo(2);     // 1 query + post check
}
```

---

## 3. S3 mock (LocalStack)

```java
@Container
static LocalStackContainer localstack = new LocalStackContainer(LOCALSTACK_IMAGE)
    .withServices(S3);

@Test
void presigned_url_generation() {
    var response = attachmentService.createPresignedUrl(userId, new PresignedRequest(
        "image.jpg", "image/jpeg", 5 * 1024 * 1024
    ));

    assertThat(response.uploadUrl()).startsWith(localstack.getEndpointOverride(S3).toString());
    assertThat(response.fileKey()).contains(userId.value());
}
```

---

## 4. 함정

자세히: [[../../signup/testing/integration-tests#9 함정]].

---

## 5. 관련

- [[testing|↑ hub]]
- [[../../signup/testing/integration-tests|↗ signup integration-tests]]
- [[../implementation/post-crud-impl]] · [[../implementation/moderation-impl]]
