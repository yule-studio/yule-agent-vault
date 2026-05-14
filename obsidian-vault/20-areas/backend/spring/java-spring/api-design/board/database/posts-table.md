---
title: "posts 테이블 — schema + 인덱스 + 정렬"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - posts-table
---

# posts 테이블 — schema + 인덱스 + 정렬

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> board 의 핵심 table. 메인 화면 / 정렬 / 검색 / counter 의 hot path.

---

## 1. Schema

```sql
-- V4__create_posts.sql
CREATE TABLE posts (
    id              CHAR(26) PRIMARY KEY,                  -- ULID
    board_id        CHAR(26) NOT NULL REFERENCES boards(id),
    author_id       CHAR(26) NOT NULL,                     -- users.id (FK 옵션)
    title           VARCHAR(200) NOT NULL,
    content         TEXT NOT NULL,                          -- markdown raw
    content_preview VARCHAR(300),                           -- 추출된 preview (메인 화면)
    status          VARCHAR(20) NOT NULL DEFAULT 'PUBLISHED',
    visibility      VARCHAR(20) NOT NULL DEFAULT 'PUBLIC',  -- PUBLIC/MEMBERS/PRIVATE

    -- counters (Redis 와 sync, 1h batch)
    view_count      INTEGER NOT NULL DEFAULT 0,
    like_count      INTEGER NOT NULL DEFAULT 0,
    comment_count   INTEGER NOT NULL DEFAULT 0,
    report_count    INTEGER NOT NULL DEFAULT 0,

    -- 정렬용
    hot_score       DOUBLE PRECISION NOT NULL DEFAULT 0,

    -- search (FTS — 10만+ 시 활성)
    content_tsv     tsvector GENERATED ALWAYS AS (
                        to_tsvector('korean', title || ' ' || content)) STORED,

    version         BIGINT NOT NULL DEFAULT 0,              -- @Version 낙관 락
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,                            -- soft delete

    CONSTRAINT chk_posts_status
        CHECK (status IN ('DRAFT', 'PUBLISHED', 'HIDDEN', 'DELETED')),
    CONSTRAINT chk_posts_visibility
        CHECK (visibility IN ('PUBLIC', 'MEMBERS', 'PRIVATE')),
    CONSTRAINT chk_posts_title_length
        CHECK (length(title) BETWEEN 1 AND 200),
    CONSTRAINT chk_posts_content_length
        CHECK (length(content) BETWEEN 1 AND 50000),
    CONSTRAINT chk_posts_counters_positive
        CHECK (view_count >= 0 AND like_count >= 0 AND
               comment_count >= 0 AND report_count >= 0)
);
```

---

## 2. 각 컬럼의 "왜"

### 2.1 `id CHAR(26)` — ULID

[[../../signup/database/id-strategy|↗ id-strategy]] 의 정책 그대로 — ULID 시간순 정렬 + URL 친화.

### 2.2 `board_id CHAR(26) NOT NULL`

**왜 필요**
- 다중 게시판 — 자유 / Q&A / 공지 분리.
- 인덱스 의 첫 컬럼 (board 별 정렬).

**왜 NULL X**
- 모든 글이 어떤 board 에 속해야 (orphan 방지).

### 2.3 `author_id CHAR(26) NOT NULL`

**왜 FK 명시 X (옵션)**
- `users.id` 참조이지만 FK 명시 시 user delete cascade 처리 복잡.
- application 단 검증.
- 옵션: `REFERENCES users(id)` 추가 후 ON DELETE SET NULL 또는 cascade.

### 2.4 `title VARCHAR(200)` + CHECK

**왜 200**
- 한국어 = ~66자.
- 너무 짧음 (50) — 정보 부족.
- 너무 김 (500) — UX (한 화면 표시).

### 2.5 `content TEXT` + CHECK 50000

**왜 TEXT (VARCHAR 아님)**
- PostgreSQL 에선 같지만 TEXT 가 의미 명확 (글 본문).
- 50000 char = 매우 긴 글도 충분 (블로그 한 편).

**안 하면 무슨 문제**
- 무제한 → DB 부담 + abuse (1MB 글).

### 2.6 `content_preview VARCHAR(300)`

**왜 별도 컬럼**
- 메인 화면 (목록) 에서 매번 markdown 파싱 부담 회피.
- INSERT/UPDATE 시점에 추출 + 저장.
- 검색 결과의 snippet 표시.

```java
// 저장 시 추출
String preview = markdownRenderer.extractPreview(content, 300);
```

### 2.7 `status VARCHAR(20)` + CHECK

**4-state**
- DRAFT — 임시 저장 (작성자만).
- PUBLISHED — 정상 (default).
- HIDDEN — 모더 또는 작성자 임시 숨김.
- DELETED — soft delete.

**왜 4-state**
- DRAFT 와 HIDDEN 의미 분리 (작성자 임시 vs 모더).
- 각 상태 별 권한 / 노출 정책 다름.

### 2.8 `visibility` (PUBLIC/MEMBERS/PRIVATE)

**왜 별도**
- status 와 다른 개념 — content 의 가시성.
- PUBLIC = 비로그인 OK.
- MEMBERS = 로그인 필수.
- PRIVATE = 작성자만 (스크랩 / 메모).

### 2.9 Counter 컬럼 (view / like / comment / report)

**왜 DB 컬럼 (Redis 만 X)**
- 정렬 / 검색 시 사용 (`ORDER BY like_count DESC`).
- Redis 장애 시 fallback.
- 1시간 batch sync.

**왜 CHECK >= 0**
- 음수 시 = 버그 / race condition. DB 가 마지막 방어.

자세히: [[../design-decisions/like-counter]] · [[../design-decisions/view-counter]].

### 2.10 `hot_score DOUBLE PRECISION`

**왜 별도 컬럼**
- 매 query 마다 계산 부담.
- 1시간 batch 계산 → 인덱스 활용.

자세히: [[../design-decisions/hot-ranking]].

### 2.11 `content_tsv tsvector GENERATED`

**왜 generated**
- INSERT/UPDATE 시 자동 계산.
- application 코드 변경 X.
- GIN 인덱스 활용 → FTS 가능.

**왜 STORED**
- `STORED` = 디스크 저장. 읽을 때 빠름.
- `VIRTUAL` = 매 read 마다 계산 (PG 미지원).

**왜 `'korean'` dictionary**
- 한국어 형태소 분석 (조사 / 어미 제거).
- 기본 (`'simple'`) = 토큰만, 정확도 ↓.
- pgroonga / mecab-ko extension 필요.

### 2.12 `version BIGINT` (@Version)

낙관 락 — 동시 수정 시 충돌 감지.

자세히: [[../../signup/database/users-table#2.11]].

### 2.13 `deleted_at TIMESTAMPTZ`

**왜 별도 (status 만으로 부족)**
- 삭제 시점 audit.
- status=DELETED 후 30일 후 hard delete 정책.

---

## 3. 인덱스

```sql
-- 메인 정렬 (board 별 hot)
CREATE INDEX ix_posts_board_hot ON posts (board_id, hot_score DESC, id DESC)
    WHERE status = 'PUBLISHED';

-- 메인 정렬 (board 별 latest)
CREATE INDEX ix_posts_board_created ON posts (board_id, created_at DESC, id DESC)
    WHERE status = 'PUBLISHED';

-- 내 글 list
CREATE INDEX ix_posts_author_created ON posts (author_id, created_at DESC)
    WHERE status = 'PUBLISHED';

-- FTS
CREATE INDEX ix_posts_content_tsv ON posts USING GIN (content_tsv);

-- Admin (HIDDEN 등 상태 별)
CREATE INDEX ix_posts_status_created ON posts (status, created_at DESC);
```

### 3.1 왜 partial index (`WHERE status = 'PUBLISHED'`)

- HIDDEN / DELETED 제외 → 인덱스 크기 ↓.
- 대부분 query 가 PUBLISHED 만.

### 3.2 왜 복합 (board_id + hot_score + id)

- 메인 화면 cursor pagination 의 정확한 매칭.
- id DESC = 같은 hot_score 시 tiebreaker.

자세히: [[../design-decisions/pagination-strategy#5 Index]].

---

## 4. 조회 패턴

| 패턴 | SQL | 빈도 |
| --- | --- | --- |
| 메인 (hot) | `WHERE board_id=? AND status='PUBLISHED' ORDER BY hot_score DESC, id DESC LIMIT 20` | 매우 잦음 |
| 메인 (latest) | `WHERE board_id=? AND status='PUBLISHED' ORDER BY created_at DESC, id DESC LIMIT 20` | 잦음 |
| 상세 | `WHERE id = ?` | 매 조회 |
| 검색 (FTS) | `WHERE content_tsv @@ tsquery AND status='PUBLISHED'` | 잦음 |
| 내 글 | `WHERE author_id=? AND status='PUBLISHED' ORDER BY created_at DESC` | 가끔 |
| Admin HIDDEN | `WHERE status='HIDDEN' ORDER BY created_at DESC` | 드뭄 |

---

## 5. JPA Entity

```java
@Entity
@Table(name = "posts")
public class PostJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "board_id", nullable = false, length = 26) private String boardId;
    @Column(name = "author_id", nullable = false, length = 26) private String authorId;
    @Column(nullable = false, length = 200) private String title;
    @Column(nullable = false, columnDefinition = "TEXT") private String content;
    @Column(name = "content_preview", length = 300) private String contentPreview;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private PostStatus status;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private PostVisibility visibility;

    @Column(name = "view_count", nullable = false) private int viewCount;
    @Column(name = "like_count", nullable = false) private int likeCount;
    @Column(name = "comment_count", nullable = false) private int commentCount;
    @Column(name = "report_count", nullable = false) private int reportCount;
    @Column(name = "hot_score", nullable = false) private double hotScore;

    @Version @Column(nullable = false) private long version;
    @Column(name = "created_at", nullable = false, updatable = false) private Instant createdAt;
    @Column(name = "updated_at", nullable = false) private Instant updatedAt;
    @Column(name = "deleted_at") private Instant deletedAt;
}
```

→ `content_tsv` 는 generated — JPA 매핑 X (DB 자동).

---

## 6. 함정 모음

### 함정 1 — counter 가 매 view 마다 UPDATE
DB lock 폭증.
→ Redis + 1h batch ([[../design-decisions/view-counter]]).

### 함정 2 — `content_preview` 안 만들고 메인 화면에서 markdown 파싱
응답 latency ↑.
→ 저장 시 추출.

### 함정 3 — partial index 없음
HIDDEN / DELETED 도 인덱스에.
→ `WHERE status = 'PUBLISHED'`.

### 함정 4 — `hot_score` 단일 인덱스 (board_id 없이)
모든 board 의 hot 한 번에 정렬 — board 별 query 비효율.
→ `(board_id, hot_score)`.

### 함정 5 — counter CHECK 누락
음수 가능 (race).
→ CHECK >= 0.

### 함정 6 — `tsvector` STORED 안 함
매 read 마다 재계산.
→ STORED.

### 함정 7 — 한국어 dictionary 없음 (`'simple'`)
정확도 ↓ (조사 / 어미).
→ pgroonga 또는 mecab-ko.

### 함정 8 — `@Version` 누락
동시 수정 시 lost update.
→ @Version 필수.

### 함정 9 — DELETED 의 content 그대로
삭제 의도 부합 X.
→ soft delete 시 content="[삭제됨]" 마스킹 + content_tsv 갱신.

### 함정 10 — `deleted_at` 없이 status 만
삭제 시점 audit X.
→ deleted_at 컬럼 추가.

### 함정 11 — `title` 길이 제한 없이
abuse (1000자 title) 시 UI 망함.
→ CHECK 1-200.

### 함정 12 — `content` 제한 없이
DB 부담 + abuse.
→ CHECK 1-50000.

---

## 7. 관련

- [[database|↑ hub]]
- [[../domain-model/post-aggregate]] — Post 도메인 (todo)
- [[../design-decisions/hot-ranking]] · [[../design-decisions/like-counter]] · [[../design-decisions/view-counter]]
- [[../design-decisions/search-strategy]] · [[../design-decisions/pagination-strategy]]
- [[comments-table]] · [[likes-tables]]
