---
title: "PostVisibility — PUBLIC / MEMBERS / PRIVATE"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - enums
  - visibility
---

# PostVisibility — PUBLIC / MEMBERS / PRIVATE

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ enums hub]]**

---

## 1. 코드

```java
public enum PostVisibility {
    PUBLIC,        // 누구나 (비로그인 포함)
    MEMBERS,       // 로그인 한 user 만
    PRIVATE;       // 작성자만 (스크랩 / 메모)

    public boolean canView(UserContext ctx, UserId authorId) {
        return switch (this) {
            case PUBLIC -> true;
            case MEMBERS -> ctx.isAuthenticated();
            case PRIVATE -> ctx.userId().equals(authorId);
        };
    }
}
```

---

## 2. status (PostStatus) 와의 차이

| | PostStatus | PostVisibility |
| --- | --- | --- |
| 의미 | lifecycle (생성 → 폐기) | 가시성 (누가 봄) |
| 변경 빈도 | 가끔 | 거의 X |
| 예 | PUBLISHED / DELETED | PUBLIC / MEMBERS |

**둘 다 필요**:
- status=PUBLISHED + visibility=PRIVATE = 정상 발행이지만 작성자만.
- status=DELETED 면 visibility 무관 (조회 X).

---

## 3. 함정

### 함정 1 — visibility 검증 누락 (status 만)
PRIVATE 글이 모두에게 노출.
→ 조회 시 둘 다 검증.

### 함정 2 — Comment 의 visibility 무시
PRIVATE post 의 댓글 다른 user 가 봄.
→ 댓글 조회 시 post.visibility 검증.

### 함정 3 — search 결과에 PRIVATE 포함
→ WHERE visibility = 'PUBLIC' (또는 viewer = author).

---

## 4. 관련

- [[enums|↑ hub]]
- [[post-status]]
- [[../security/authentication-authorization]] (todo)
