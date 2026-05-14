---
title: "인증 / 인가 — 권한 매트릭스 (board)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:38:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - security
  - authn
  - authz
---

# 인증 / 인가 — 권한 매트릭스 (board)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[security|↑ security hub]]**

> board endpoint 의 권한. signup 의 SecurityConfig 위에 board 매트릭스 추가.

---

## 1. Endpoint 권한 매트릭스

### 1.1 게시글

| Endpoint | anonymous | USER | ADMIN | 비고 |
| --- | --- | --- | --- | --- |
| GET /boards/{id}/posts | ✅ (board public) | ✅ | ✅ | visibility=PRIVATE 제외 |
| GET /posts/{id} | ✅ (PUBLIC) | ✅ | ✅ | HIDDEN/DELETED 는 404 |
| POST /boards/{id}/posts | ❌ | ✅ | ✅ | |
| PATCH /posts/{id} | ❌ | ✅ 작성자 | ✅ | 작성자 검증 |
| DELETE /posts/{id} | ❌ | ✅ 작성자 | ✅ | |
| PATCH /admin/posts/{id}/status | ❌ | ❌ | ✅ | 모더 (hide/restore) |

### 1.2 댓글

| Endpoint | anonymous | USER | ADMIN | 비고 |
| --- | --- | --- | --- | --- |
| GET /posts/{id}/comments | ✅ | ✅ | ✅ | |
| POST /posts/{id}/comments | ❌ | ✅ | ✅ | |
| PATCH /comments/{id} | ❌ | ✅ 작성자 | ✅ | |
| DELETE /comments/{id} | ❌ | ✅ 작성자 | ✅ | |

### 1.3 좋아요 / 북마크 / 신고 / 차단

| Endpoint | 권한 |
| --- | --- |
| POST/DELETE /posts/{id}/like | USER+ |
| POST/DELETE /posts/{id}/bookmark | USER+ |
| POST /posts/{id}/report | USER+ |
| POST /users/{id}/block | USER+ |

### 1.4 Admin

| Endpoint | 권한 |
| --- | --- |
| GET /admin/reports | ADMIN |
| PATCH /admin/reports/{id} | ADMIN |
| GET /admin/posts (HIDDEN list) | ADMIN |
| PATCH /admin/posts/{id}/status | ADMIN |

---

## 2. SecurityConfig

```java
.requestMatchers(HttpMethod.GET,
    "/api/v1/boards/*/posts",
    "/api/v1/posts/*",
    "/api/v1/posts/*/comments"
).permitAll()
.requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
.anyRequest().authenticated()
```

자세히: [[../../signup/security/authentication-authorization]].

---

## 3. 작성자 검증

```java
@PreAuthorize("@boardSecurity.isAuthor(#postId, authentication)")
@PatchMapping("/posts/{postId}")
public PostResponse updatePost(@PathVariable PostId postId, ...) { ... }
```

```java
@Component("boardSecurity")
public class BoardSecurityExpressions {
    private final PostRepository posts;

    public boolean isAuthor(PostId postId, Authentication auth) {
        var user = (AuthUser) auth.getPrincipal();
        return posts.findById(postId)
            .map(p -> p.isOwnedBy(user.id()))
            .orElse(false);
    }
}
```

→ Service 안 검증보다 명시적.

---

## 4. visibility 검증

```java
public Post viewPost(PostId postId, AuthUser viewer) {
    var post = posts.findById(postId)
        .orElseThrow(() -> new NotFoundException("post"));

    if (post.status() == DELETED) throw new NotFoundException("post");
    if (post.status() == HIDDEN && !viewer.isAdmin() && !post.isOwnedBy(viewer.id()))
        throw new NotFoundException("post");

    if (!post.visibility().canView(viewer, post.authorId()))
        throw new ForbiddenException();

    return post;
}
```

---

## 5. 함정

### 함정 1 — 작성자 검증 누락
다른 user 의 글 수정 / 삭제 가능.
→ @PreAuthorize 또는 service 안 검증.

### 함정 2 — DELETED 글 노출
soft delete 후 status filter 누락.
→ WHERE status != 'DELETED'.

### 함정 3 — HIDDEN 글 작성자가 못 봄
admin 모더 후 작성자가 자기 글 안 보임 → 항의.
→ 작성자 / admin 은 HIDDEN 도 access.

### 함정 4 — Admin endpoint 가 USER 권한
권한 escalation.
→ hasRole('ADMIN').

### 함정 5 — visibility 검증 시점 race
post 가 PRIVATE 으로 변경되는 동안 다른 user 가 조회 — 잠시 노출.
→ 트랜잭션 격리 (READ_COMMITTED 충분).

### 함정 6 — PreAuthorize 의 method 의 DB 호출
매 요청마다 post lookup → 부담 ↑.
→ cache (옵션) 또는 service 안 검증.

---

## 6. 관련

- [[security|↑ hub]]
- [[../../signup/security/authentication-authorization]] — 베이스
- [[../enums/post-status]] · [[../enums/post-visibility]]
- [[block-filter]]
