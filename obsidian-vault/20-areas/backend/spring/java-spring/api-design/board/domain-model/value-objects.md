---
title: "Value Objects — PostId / CommentId / TargetId"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - domain-model
  - value-object
---

# Value Objects — PostId / CommentId / TargetId

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ domain-model hub]]**

> board 도메인의 ID VO. signup 의 [[../../signup/domain-model/value-objects|VO 패턴]] 따름.

---

## 1. ID VO 4종

```java
// PostId
public record PostId(@JsonValue String value) {
    public PostId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("invalid PostId (ULID 26 chars): " + value);
    }
    public static PostId generate() {
        return new PostId(UlidCreator.getMonotonicUlid().toString());
    }
}

// CommentId
public record CommentId(@JsonValue String value) {
    public CommentId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("invalid CommentId");
    }
    public static CommentId generate() {
        return new CommentId(UlidCreator.getMonotonicUlid().toString());
    }
}

// BoardId
public record BoardId(@JsonValue String value) {
    public BoardId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("invalid BoardId");
    }
}

// ReportId
public record ReportId(@JsonValue String value) {
    public ReportId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("invalid ReportId");
    }
    public static ReportId generate() {
        return new ReportId(UlidCreator.getMonotonicUlid().toString());
    }
}
```

---

## 2. TargetId — Post 또는 Comment

신고 / 좋아요 같은 곳에서 "post 또는 comment" 를 가리킬 때:

```java
public sealed interface TargetId {
    String value();
    TargetType type();

    static TargetId of(PostId postId) { return new PostTargetId(postId); }
    static TargetId of(CommentId commentId) { return new CommentTargetId(commentId); }
}

public record PostTargetId(PostId postId) implements TargetId {
    @Override public String value() { return postId.value(); }
    @Override public TargetType type() { return TargetType.POST; }
}

public record CommentTargetId(CommentId commentId) implements TargetId {
    @Override public String value() { return commentId.value(); }
    @Override public TargetType type() { return TargetType.COMMENT; }
}

public enum TargetType { POST, COMMENT }
```

### 2.1 왜 sealed interface

- POST / COMMENT 만 — 새 type (USER 같은) 추가 시 명시.
- pattern matching 친화.

### 2.2 왜 별도 record (`PostTargetId`, `CommentTargetId`)

- 타입 안전성 — `reportPost(postId)` vs `reportComment(commentId)` 명시.
- application 의 `if (target.type() == POST) ...` 분기 명확.

### 2.3 왜 enum (TargetType) 도

- DB schema 의 `target_type VARCHAR(20)` 와 매핑.
- 단순 분기에 enum 이 sealed 보다 자연스러움.

---

## 3. 다른 VO

### 3.1 PostStatus / CommentStatus / ReportStatus

각자 enum + 자기 노트.

자세히: [[../enums/enums]] (todo).

### 3.2 PostVisibility

```java
public enum PostVisibility { PUBLIC, MEMBERS, PRIVATE }
```

### 3.3 ReportReason

```java
public enum ReportReason {
    SPAM, INAPPROPRIATE, HARASSMENT, HATE_SPEECH,
    VIOLENCE, COPYRIGHT, PERSONAL_INFO, OTHER
}
```

### 3.4 Cursor (페이지네이션)

```java
public record Cursor(Instant createdAt, String id) {
    public String encode() { /* base64 json */ }
    public static Cursor decode(String s) { /* */ }
    public static Cursor first() { return new Cursor(Instant.MAX, "Z"); }
}
```

자세히: [[../design-decisions/pagination-strategy#3 Cursor 구현]].

---

## 4. signup VO 와의 관계

board 가 `UserId`, `Email`, `PhoneNumber` 등 signup VO 그대로 사용:

```java
// signup 의 UserId
public record UserId(String value) { /* ... */ }

// board 의 Post 가 UserId 참조
public final class Post {
    private final UserId authorId;        // signup VO
    // ...
}
```

→ VO 재사용. import path 만 다름.

자세히: [[../../signup/domain-model/value-objects|↗ signup VO]].

---

## 5. 함정

### 함정 1 — PostId / CommentId 가 String
타입 안전성 X.
→ VO 강제.

### 함정 2 — TargetId 가 String + type 컬럼만
application 단 분기 매번.
→ sealed interface.

### 함정 3 — UserId vs PostId 혼동
`process(String authorId, String postId)` 매개변수 헷갈림.
→ VO 로 명시.

### 함정 4 — `@JsonValue` 누락
응답에 `{ "id": { "value": "01HQ..." } }` wrapper.
→ `@JsonValue` 단일 String.

### 함정 5 — VO 가 mutable (final field 없음)
HashSet / HashMap 이상.
→ record 사용.

### 함정 6 — ULID 길이 검증 누락
26 char 외 통과.
→ length 검증.

### 함정 7 — `Cursor` 가 sort 다른 시 깨짐
sort=latest → sort=hot 변경 시 cursor invalid.
→ cursor 안 sort key 포함.

---

## 6. 관련

- [[domain-model|↑ hub]]
- [[../../signup/domain-model/value-objects|↗ signup VO]] — 동일 패턴
- [[../enums/enums]] (todo)
- [[../database/posts-table]] — DB 매핑
