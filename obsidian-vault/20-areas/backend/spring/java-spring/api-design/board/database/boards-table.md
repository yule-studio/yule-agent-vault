---
title: "boards 테이블 — 게시판 종류"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - boards-table
---

# boards 테이블 — 게시판 종류

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 게시판 종류 (자유 / Q&A / 공지). posts 의 FK 대상.

---

## 1. Schema

```sql
-- V1__create_boards.sql
CREATE TABLE boards (
    id              CHAR(26) PRIMARY KEY,
    code            VARCHAR(50) NOT NULL UNIQUE,           -- 'free', 'qna', 'notice'
    name            VARCHAR(100) NOT NULL,                  -- '자유게시판', 'Q&A'
    description     TEXT,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'PUBLIC',
    display_mode    VARCHAR(20) NOT NULL DEFAULT 'NICKNAME',
    allow_anonymous BOOLEAN NOT NULL DEFAULT false,
    require_login_to_view BOOLEAN NOT NULL DEFAULT false,
    require_login_to_post BOOLEAN NOT NULL DEFAULT true,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    use_yn          VARCHAR(1) NOT NULL DEFAULT 'Y',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_boards_visibility
        CHECK (visibility IN ('PUBLIC', 'MEMBERS', 'PRIVATE')),
    CONSTRAINT chk_boards_display_mode
        CHECK (display_mode IN ('NICKNAME', 'ANONYMOUS', 'REAL_NAME'))
);

CREATE INDEX ix_boards_active ON boards (sort_order, id) WHERE use_yn = 'Y';
```

---

## 2. 컬럼 "왜"

### 2.1 `code` UNIQUE
- URL friendly (`/boards/free` vs `/boards/01HQ...`).
- 환경 별 동일 code 유지 (dev / prod).

### 2.2 `display_mode`
- NICKNAME / ANONYMOUS / REAL_NAME — [[../design-decisions/anonymity-policy]].

### 2.3 `require_login_to_view` vs `to_post`
- view = 비로그인 OK 인 공개 게시판 / 회원 전용 게시판.
- post = 글쓰기는 항상 로그인 (default).

### 2.4 `use_yn`
- 폐기된 게시판도 row 보존 (옛 글 audit). 검색은 active 만.

---

## 3. Seed 데이터

```sql
-- V1_1__seed_initial_boards.sql
INSERT INTO boards (id, code, name, description, sort_order) VALUES
('01HZBOARD01FREE',   'free',   '자유게시판',     '자유로운 이야기',    1),
('01HZBOARD02QNA',    'qna',    'Q&A',           '질문과 답변',        2),
('01HZBOARD03NOTICE', 'notice', '공지사항',       '운영자 공지 only',   3),
('01HZBOARD04TIP',    'tip',    '꿀팁',          '유용한 정보',        4);
```

---

## 4. JPA Entity

```java
@Entity @Table(name = "boards")
public class BoardJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(nullable = false, length = 50, unique = true) private String code;
    @Column(nullable = false, length = 100) private String name;
    @Column(columnDefinition = "TEXT") private String description;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private BoardVisibility visibility;

    @Enumerated(EnumType.STRING)
    @Column(name = "display_mode", nullable = false, length = 20) private DisplayMode displayMode;

    @Column(name = "allow_anonymous", nullable = false) private boolean allowAnonymous;
    @Column(name = "require_login_to_view", nullable = false) private boolean requireLoginToView;
    @Column(name = "require_login_to_post", nullable = false) private boolean requireLoginToPost;

    @Column(name = "sort_order", nullable = false) private int sortOrder;
    @Column(name = "use_yn", nullable = false, length = 1) private String useYn = "Y";
    @Column(name = "created_at", nullable = false) private Instant createdAt;
}
```

---

## 5. 함정

### 함정 1 — `code` UNIQUE 누락
같은 code 두 게시판 → URL 매핑 모호.

### 함정 2 — Hard delete 게시판
옛 글 / 댓글의 board_id 가 dangling.
→ use_yn='N'.

### 함정 3 — `name` 변경 무제한
URL bookmark 영향 X (code 가 URL) but 사용자 혼동.
→ admin only + 변경 audit.

### 함정 4 — `sort_order` 없음
게시판 list 가 random.
→ sort_order INTEGER.

---

## 6. 관련

- [[database|↑ hub]]
- [[posts-table]] — board_id FK
- [[../design-decisions/anonymity-policy]] — display_mode
