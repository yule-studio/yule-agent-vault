---
title: "board §3 — 요구사항 + 완료 조건"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - requirements
---

# board §3 — 요구사항 + 완료 조건

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 — 8 기능 영역의 Acceptance Criteria |

**[[board|↑ hub]]**  ·  ← [[prerequisites]]  ·  → [[design-decisions/design-decisions]]

> 본 board recipe 가 "완료" 인지 판단하는 **객관적 완료 조건** (Acceptance Criteria).

---

## 1. 기능 요구 — 8 영역

| # | 영역 | 요구 |
| --- | --- | --- |
| 1 | 게시글 CRUD | 작성 / 조회 / 수정 / 삭제 (soft) + 작성자 검증 |
| 2 | 댓글 | 작성 / 조회 / 수정 / 삭제 (soft) + 대댓글 (2-level) |
| 3 | 좋아요 / 북마크 | toggle + counter (eventual consistency) |
| 4 | 검색 / 정렬 | 검색 + 정렬 (latest / hot / top) + cursor pagination |
| 5 | 카테고리 / 태그 | category 분류 + tag 자유 부착 |
| 6 | 첨부 파일 | S3 presigned URL + 이미지 / 동영상 |
| 7 | 신고 / 모더레이션 | 신고 + 자동 hide (5회) + admin review |
| 8 | 차단 사용자 | block list + 조회 시 filter |

---

## 2. 완료 조건 — 각 영역별 Acceptance Criteria

### 2.1 게시글 CRUD

- [ ] **AC-1**: 인증된 user 가 `POST /boards/{boardId}/posts` 로 글 작성 시 `status=PUBLISHED`, `author_id` 자동.
- [ ] **AC-2**: 작성자 본인만 `PATCH /posts/{postId}` 수정 — 다른 user 시 403.
- [ ] **AC-3**: `DELETE /posts/{postId}` 가 soft delete (`status=DELETED`) — row 삭제 X.
- [ ] **AC-4**: 삭제된 글의 댓글은 보존되지만 글 자체는 목록 / 조회 X.
- [ ] **AC-5**: `content` 에 `<script>` 입력 시 sanitize — 다른 user 화면에 실행 X.
- [ ] **AC-6**: 비공개 board 의 글은 권한 없는 user 가 404.
- [ ] **AC-7**: 동시 작성 (race) 시 — 정상 처리 (DB UNIQUE 위반 시 다른 ID로 retry 또는 명확 에러).

### 2.2 댓글 / 대댓글

- [ ] **AC-8**: `POST /posts/{postId}/comments` — flat comment 작성.
- [ ] **AC-9**: `POST /comments/{parentId}/reply` — 대댓글 (parent_id 매핑).
- [ ] **AC-10**: 대댓글에 다시 대대댓글 시도 → 자동 부모 comment 에 평탄화 (2-level 강제).
- [ ] **AC-11**: comment 삭제 시 → `status=DELETED` + content="[삭제된 댓글]" 마스킹 (대댓글 보존).
- [ ] **AC-12**: 댓글 list 조회 시 tree 구조 (parent → children 매핑).
- [ ] **AC-13**: post 의 comment_count 가 댓글 INSERT/DELETE 시 자동 업데이트.

### 2.3 좋아요 / 북마크

- [ ] **AC-14**: 같은 user 가 같은 post 에 두 번 좋아요 시도 → 첫 번째만 INSERT, 두 번째 silent (또는 idempotent).
- [ ] **AC-15**: 좋아요 취소 시 row DELETE + counter -1.
- [ ] **AC-16**: counter (post.like_count) 가 Redis 와 DB sync — 1시간 batch 보정.
- [ ] **AC-17**: 북마크는 좋아요와 동일 패턴, 별도 테이블.
- [ ] **AC-18**: 좋아요 / 북마크 endpoint 에 rate limit (분당 60회 — abuse 방어).

### 2.4 검색 / 정렬 / 페이지네이션

- [ ] **AC-19**: `GET /posts?q={query}` — title + content ILIKE 검색.
- [ ] **AC-20**: `sort=latest` (createdAt DESC), `sort=hot` (시간 감쇠 score), `sort=top` (like_count DESC).
- [ ] **AC-21**: cursor-based pagination (`cursor=...&limit=20`) — 무한 스크롤 호환.
- [ ] **AC-22**: hot 정렬의 score 가 시간 따라 감쇠 (Reddit 식 — `log10(likes) + (createdAt - epoch) / 45000`).
- [ ] **AC-23**: 검색 결과에 차단한 user 의 글 제외.
- [ ] **AC-24**: HIDDEN / DELETED 글 검색 결과 제외.

### 2.5 카테고리 / 태그

- [ ] **AC-25**: board 마다 category 정의 (예: 자유 / 일상 / 정보).
- [ ] **AC-26**: 글 작성 시 category 선택 (board 의 category list 안에서만).
- [ ] **AC-27**: tag 는 자유 입력 — 첫 사용 시 자동 생성.
- [ ] **AC-28**: `GET /posts?tag={tag}&boardId=...` — tag 필터 + board 필터.
- [ ] **AC-29**: 인기 태그 (`GET /tags/popular`) — 최근 7일 사용 count DESC.

### 2.6 첨부 파일

- [ ] **AC-30**: `POST /posts/attachments/presigned` — file metadata 받아서 S3 presigned URL 발급.
- [ ] **AC-31**: 파일 크기 max 50MB / type whitelist (image/video).
- [ ] **AC-32**: 글 작성 시 attachmentKeys 배열 포함 → 미완료 업로드 자동 cleanup.
- [ ] **AC-33**: 글 삭제 시 첨부 파일도 S3 에서 삭제 (또는 GDPR 30일 후).
- [ ] **AC-34**: presigned URL TTL 5분 — 만료 후 업로드 시도 → 403.

### 2.7 신고 / 모더레이션 / 차단

- [ ] **AC-35**: `POST /posts/{postId}/report { reason }` — 신고 INSERT.
- [ ] **AC-36**: 같은 user 가 같은 target 에 중복 신고 → 두 번째 silent (또는 409).
- [ ] **AC-37**: target 의 신고 수 ≥ 5 → 자동 `status=HIDDEN` + 작성자 알림.
- [ ] **AC-38**: ADMIN 의 `PATCH /admin/posts/{postId}/status` 로 수동 hide / restore.
- [ ] **AC-39**: `POST /users/{userId}/block` → block list 에 추가.
- [ ] **AC-40**: 차단한 user 의 글 / 댓글 조회 시 자동 제외.

### 2.8 알림 (옵션)

- [ ] **AC-41**: 글 작성자가 좋아요 받으면 알림 outbox INSERT.
- [ ] **AC-42**: 글 작성자가 댓글 받으면 알림 outbox INSERT.
- [ ] **AC-43**: 댓글 작성자가 대댓글 받으면 알림 (자기 글 X).
- [ ] **AC-44**: 알림 발송 = AFTER_COMMIT — 트랜잭션 실패 시 미발송.
- [ ] **AC-45**: 사용자가 알림 OFF 설정 시 outbox INSERT X.

---

## 3. 비기능 요구

### 3.1 성능

| 메트릭 | 목표 (p99) |
| --- | --- |
| 게시글 목록 조회 | < 200ms |
| 게시글 상세 조회 | < 100ms (cache hit) / < 300ms (miss) |
| 게시글 작성 | < 500ms |
| 댓글 작성 | < 200ms |
| 좋아요 toggle | < 100ms (Redis HIT) |
| 검색 (DB ILIKE) | < 500ms (1만 posts) |
| 검색 (FTS) | < 200ms (100만 posts) |

### 3.2 보안

- [ ] 모든 mutation endpoint 인증.
- [ ] 작성자 검증 (`PATCH/DELETE`) — 본인 또는 ADMIN 만.
- [ ] XSS sanitize (content / title) — OWASP HTML Sanitizer.
- [ ] CSRF (cookie 인증 시) — JWT header 이므로 자연 방어.
- [ ] Rate limit — 글/댓글 작성 분당 5회 / user.
- [ ] 첨부 파일 type whitelist (image/jpeg, image/png, video/mp4 등).
- [ ] presigned URL 5분 TTL.

### 3.3 정합성

- [ ] post.comment_count 가 comments 테이블 count 와 일치.
- [ ] post.like_count 가 post_likes count 와 일치 (1시간 grace).
- [ ] post 의 board_id FK 유효.
- [ ] comment 의 post_id FK 유효 + parent_id FK 유효 (자체 참조).
- [ ] 차단된 user 의 글이 조회 결과에서 제외 (consistent).

### 3.4 운영

- [ ] 모든 endpoint 의 메트릭 (성공 / 실패 / latency).
- [ ] HIDDEN / DELETED 글의 조회 시도 메트릭 (404 / 403).
- [ ] 신고 발생 알람 (admin Slack).
- [ ] 첨부 storage 용량 모니터링.
- [ ] Cache hit rate (Redis) 메트릭.

---

## 4. 비기능 — 운영 정책

| 정책 | 값 |
| --- | --- |
| 게시글 max length | content 50,000 char / title 200 char |
| 댓글 max length | 5,000 char |
| 첨부 max size | 50 MB / file |
| 첨부 max count | 10 / post |
| 글 작성 rate limit | 5 / 분 (user) |
| 댓글 작성 rate limit | 10 / 분 (user) |
| 좋아요 toggle rate limit | 60 / 분 (user) |
| 자동 hide threshold | 5 신고 |
| Soft delete grace | 30일 — 후 hard delete (옵션) |
| Search query length | max 100 char |
| 페이지네이션 limit | max 50 / page |

---

## 5. 우선순위 — 출시 시 무엇부터

### 🔴 Must (출시 전 필수)

- 게시글 CRUD + 작성자 검증 + soft delete
- 댓글 (flat — 대댓글 후순위)
- 좋아요 (북마크 후순위)
- 카테고리 (단순 — board 1개)
- 검색 (DB ILIKE — 정확도 후순위)
- cursor pagination
- XSS sanitize
- 신고 (자동 hide 후순위 — 수동 모더만)

### 🟡 권장 (sprint 안)

- 대댓글 (2-level)
- 북마크
- 태그
- 다중 게시판
- 첨부 파일
- 인기 글 (hot 정렬)
- 자동 hide (5회)
- 차단 사용자

### 🟢 옵션 (안정 후)

- FTS / Elasticsearch
- 좋아요 / 댓글 알림 (FCM)
- 익명 옵션
- 임시 저장 (DRAFT)
- 글 통계 (view / share)

---

## 6. 관련

- [[board|↑ hub]]
- [[prerequisites]] — 이전 (§2)
- [[design-decisions/design-decisions]] — 다음 (§4)
- [[testing/test-scenarios]] — 시나리오 표 (todo)
- [[implementation-order]] — 단계별 PR
