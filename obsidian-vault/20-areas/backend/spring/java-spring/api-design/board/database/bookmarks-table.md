---
title: "post_bookmarks 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - bookmarks
---

# post_bookmarks 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 북마크 — 좋아요와 같은 패턴, 별도 의미 (저장 vs 반응).

---

## 1. Schema

```sql
-- V11__create_post_bookmarks.sql
CREATE TABLE post_bookmarks (
    user_id    CHAR(26) NOT NULL,
    post_id    CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, post_id)
);

CREATE INDEX ix_post_bookmarks_user_created ON post_bookmarks (user_id, created_at DESC);
```

---

## 2. 왜 좋아요와 별도

**의미 분리**
- 좋아요 = "공감 / 인기" (counter 노출).
- 북마크 = "나중에 보기" (개인 자료).

**UX 차이**
- 좋아요는 모든 사용자에게 counter 표시.
- 북마크는 본인만 — `/me/bookmarks` 만.

**counter X**
- post.bookmark_count 컬럼 없음 (의미 X).
- 좋아요와 다른 점.

---

## 3. 조회 — 내 북마크 list

```sql
SELECT p.* FROM posts p
JOIN post_bookmarks b ON b.post_id = p.id
WHERE b.user_id = ?
  AND p.status = 'PUBLISHED'
ORDER BY b.created_at DESC
LIMIT 20;
```

→ `created_at` 인덱스 활용.

---

## 4. 함정

### 함정 1 — bookmark_count 컬럼 추가
의미 없음 + counter overhead.
→ 그냥 없음.

### 함정 2 — 비공개 post 의 북마크
post visibility 변경 후 — 북마크는 그대로.
→ 조회 시 visibility 검증.

### 함정 3 — DELETED post 의 북마크
사용자 화면에 잠시 표시.
→ JOIN + status filter.

---

## 5. 관련

- [[likes-tables]] — 같은 패턴
- [[database|↑ hub]]
