---
title: "post_attachments 테이블 — S3 file key 매핑"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - attachments
---

# post_attachments 테이블 — S3 file key 매핑

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 첨부 파일의 metadata — 실제 file 은 S3.

---

## 1. Schema

```sql
-- V7__create_post_attachments.sql
CREATE TABLE post_attachments (
    id          CHAR(26) PRIMARY KEY,
    post_id     CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    file_key    VARCHAR(500) NOT NULL UNIQUE,        -- S3 key
    file_name   VARCHAR(255) NOT NULL,                -- 원본 file name
    content_type VARCHAR(100) NOT NULL,
    size_bytes  BIGINT NOT NULL,
    sort_order  INTEGER NOT NULL DEFAULT 0,
    width       INTEGER,                              -- 이미지 너비 (옵션)
    height      INTEGER,                              -- 이미지 높이 (옵션)
    duration    INTEGER,                              -- 동영상 초 (옵션)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_attachments_size_positive
        CHECK (size_bytes > 0 AND size_bytes <= 52428800)    -- 50MB max
);

CREATE INDEX ix_post_attachments_post ON post_attachments (post_id, sort_order);
```

---

## 2. 컬럼 "왜"

### 2.1 `file_key VARCHAR(500) UNIQUE`

- S3 의 object key — `{userId}/{yearMonth}/{ulid}-{name}`.
- UNIQUE — 같은 key 두 row X.
- 500 char = path + name 충분.

### 2.2 `file_name` (원본)

- S3 key 는 sanitize 됨 (path traversal 방어).
- 사용자 download 시 원본 name 으로 (`Content-Disposition`).

### 2.3 `content_type` + size

- 응답 시 사용자에게 표시 (이미지 / 동영상 구분).
- abuse 분석 (50MB 동영상 1000개 = storage 폭증).

### 2.4 `width / height / duration`

- 이미지: lazy load placeholder (LQIP) 의 정확한 비율.
- 동영상: 미리보기 표시.
- 옵션 (없으면 NULL).

### 2.5 `sort_order`

- post 안 첨부의 표시 순서.
- 사용자가 drag & drop 으로 재정렬.

### 2.6 ON DELETE CASCADE (post)

- post hard delete 시 attachment row 자동 삭제.
- 단 — S3 의 실제 file 은 별도 cleanup 필요.

---

## 3. S3 cleanup 흐름

```mermaid
flowchart TD
    A[post hard delete] --> B[ON DELETE CASCADE<br/>post_attachments row 삭제]
    B --> C{S3 file 도 삭제?}
    C -->|즉시| D[S3 DeleteObject<br/>(application 트리거)]
    C -->|지연| E[S3 lifecycle<br/>30일 후 자동 삭제]

    F[orphan 업로드<br/>presigned 후 post 안 만듦] --> G[S3 lifecycle<br/>1일 후 자동 삭제]

    style D fill:#fecaca
    style E fill:#fef3c7
```

### 3.1 본 vault — Hybrid

- post 삭제 시 즉시 `S3 DeleteObject` (application).
- orphan (presigned 후 post 안 만든 file) — S3 lifecycle 1일.
- 강제 cleanup 누락 시 30일 후 lifecycle 안전망.

자세히: [[../design-decisions/attachment-storage#5 cleanup]].

---

## 4. Storage 비용 모니터링

```sql
-- 사용자별 storage 사용량
SELECT
    p.author_id,
    COUNT(a.id) AS file_count,
    SUM(a.size_bytes) AS total_bytes
FROM post_attachments a
JOIN posts p ON p.id = a.post_id
WHERE p.status != 'DELETED'
GROUP BY p.author_id
ORDER BY total_bytes DESC;
```

→ abuse 감지 (한 user 가 100GB).

---

## 5. 함정

### 함정 1 — file_key 없이 application 단 매핑
S3 의 실제 file 추적 X → orphan 영원.
→ DB 의 file_key UNIQUE 보존.

### 함정 2 — ON DELETE CASCADE 없이
post 삭제 후 attachment row 좀비.
→ CASCADE.

### 함정 3 — S3 cleanup 없음
DB row 삭제했는데 S3 file 영구 → storage 비용 폭증.
→ application 즉시 + lifecycle 안전망.

### 함정 4 — size CHECK 없음
abuse (1GB file).
→ CHECK 50MB max.

### 함정 5 — file_name 사용자 입력 그대로 download
`../../etc/passwd` 같은 path traversal.
→ sanitize 또는 ulid prefix.

### 함정 6 — width / height 안 저장
lazy load layout shift (이미지 로드 후 화면 점프).
→ INSERT 시 추출 (서버에서 image header 분석).

### 함정 7 — sort_order 없음
사용자 의도 순서 X.
→ sort_order INTEGER.

---

## 6. 관련

- [[database|↑ hub]]
- [[../design-decisions/attachment-storage]] — S3 presigned 정책
- [[posts-table]] — post 의 첨부
- [[../implementation/attachment-impl]] (todo)
