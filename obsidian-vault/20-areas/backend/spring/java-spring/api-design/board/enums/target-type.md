---
title: "TargetType — POST / COMMENT"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:18:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - enums
  - target
---

# TargetType — POST / COMMENT

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ enums hub]]**

> Report / Like 의 target 종류. post 또는 comment.

---

## 1. 코드

```java
public enum TargetType {
    POST,
    COMMENT
}
```

---

## 2. 사용처

| Table | target_type 사용 |
| --- | --- |
| `reports` | POST 또는 COMMENT 신고 |
| (옵션) 통합 `likes` | POST 또는 COMMENT 좋아요 |

**본 vault**: `post_likes` / `comment_likes` 별도 — 단순성. TargetType 은 reports 만.

---

## 3. 왜 2개 만 (USER 같은 추가 X)

- 게시판의 좋아요 / 신고 대상 = post / comment 만.
- user 자체는 [[../database/user-blocks-table|차단]] 으로 별도 처리.
- 향후 추가 시 enum 확장 OK (이름 변경 X).

---

## 4. application 분기

```java
public void hide(TargetId target) {
    switch (target.type()) {
        case POST -> postRepo.hide(target.toPostId());
        case COMMENT -> commentRepo.hide(target.toCommentId());
    }
}
```

→ pattern matching 깔끔.

---

## 5. 함정

### 함정 1 — String 사용 (enum X)
오타 / 일관성 X.
→ enum + CHECK.

### 함정 2 — TargetId 가 String + type 컬럼 분리
타입 안전성 X.
→ sealed TargetId ([[../domain-model/value-objects#2]]).

---

## 6. 관련

- [[enums|↑ hub]]
- [[../database/reports-table]]
- [[../domain-model/value-objects#2 TargetId]]
