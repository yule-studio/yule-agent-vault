---
title: "Pagination / Tree 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:26:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - pitfalls
  - pagination
---

# Pagination / Tree 함정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — Offset 깊은 page
page 100 = OFFSET 2000.
→ cursor.

## 함정 2 — Cursor single column
같은 created_at 시 모호.
→ (created_at, id) tuple.

## 함정 3 — Cursor 인코딩 X
사용자가 임의 변경 가능.
→ base64.

## 함정 4 — Cursor type 검증 X
sort=hot cursor 를 sort=latest 에 사용.
→ cursor 안 sort key.

## 함정 5 — Limit 무제한
abuse.
→ max 50.

## 함정 6 — Composite index 없음
정렬 풀스캔.
→ (board_id, sort_key DESC, id DESC).

## 함정 7 — Partial index 없음
HIDDEN / DELETED 도 인덱스.
→ `WHERE status = 'PUBLISHED'`.

## 함정 8 — Comment tree N+1
댓글마다 자식 query.
→ 한 query + Map 그룹화.

## 함정 9 — Block filter 후 row 부족
20 fetch 후 18 표시.
→ limit+1 + over-fetch buffer.

## 함정 10 — Cursor 변경 가능한 컬럼
updated_at cursor — 글 수정 시 깨짐.
→ created_at 또는 immutable.

---

## 관련

- [[../design-decisions/pagination-strategy]] · [[../design-decisions/hot-ranking]]
- [[../design-decisions/comment-structure]]
- [[pitfalls|↑ hub]]
