---
title: "categories / tags 테이블 — 분류 / 태그"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - taxonomy
---

# categories / tags 테이블 — 분류 / 태그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> Category (board 안 sub-classification) + Tag (자유 부착, M:N).

---

## 1. Schema

```sql
-- V2__create_categories.sql
CREATE TABLE categories (
    id          CHAR(26) PRIMARY KEY,
    board_id    CHAR(26) NOT NULL REFERENCES boards(id),
    code        VARCHAR(50) NOT NULL,
    name        VARCHAR(100) NOT NULL,
    sort_order  INTEGER NOT NULL DEFAULT 0,
    use_yn      VARCHAR(1) NOT NULL DEFAULT 'Y',

    UNIQUE (board_id, code)
);

CREATE INDEX ix_categories_board_active ON categories (board_id, sort_order)
    WHERE use_yn = 'Y';

-- V5__create_post_categories.sql
CREATE TABLE post_categories (
    post_id     CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    category_id CHAR(26) NOT NULL REFERENCES categories(id),
    PRIMARY KEY (post_id, category_id)
);

CREATE INDEX ix_post_categories_category ON post_categories (category_id);

-- V3__create_tags.sql
CREATE TABLE tags (
    id          CHAR(26) PRIMARY KEY,
    name        VARCHAR(50) NOT NULL UNIQUE,
    usage_count INTEGER NOT NULL DEFAULT 0,           -- 인기 태그
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_tags_usage ON tags (usage_count DESC);
CREATE INDEX ix_tags_name_trgm ON tags USING GIN (name gin_trgm_ops);   -- 자동완성

-- V6__create_post_tags.sql
CREATE TABLE post_tags (
    post_id CHAR(26) NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  CHAR(26) NOT NULL REFERENCES tags(id),
    PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX ix_post_tags_tag ON post_tags (tag_id);
```

---

## 2. Category vs Tag — 의미 차이

| | Category | Tag |
| --- | --- | --- |
| **소속** | board 안 | 글로벌 |
| **생성** | admin 미리 정의 | 사용자 자유 입력 |
| **수량** | board 당 5-20 | 무한 (수만+) |
| **관계** | post : category = N:M (보통 1:1) | post : tag = N:M (보통 1:5) |
| **예** | "후기" / "질문" / "공지" | "맛집" / "강남" / "한식" |

---

## 3. Tag — 자동 생성

```java
@Transactional
public Set<TagId> resolveOrCreate(Set<String> tagNames) {
    Set<TagId> result = new HashSet<>();
    for (String name : tagNames) {
        var normalized = name.trim().toLowerCase();
        var existing = tags.findByName(normalized);
        if (existing.isPresent()) {
            result.add(existing.get().id());
            tags.incrementUsage(existing.get().id());
        } else {
            var newTag = Tag.create(normalized);
            tags.save(newTag);
            result.add(newTag.id());
        }
    }
    return result;
}
```

### 3.1 왜 자동 생성

- 사용자가 자유 입력 → 태그 system 활성.
- "관리자가 미리 정의" 시 사용자 시작 어려움.

### 3.2 왜 normalized (lowercase + trim)

- "맛집" / "맛집 " / "맛집" → 같은 tag.
- name UNIQUE 의 정확한 매칭.

### 3.3 usage_count

- 인기 태그 정렬 — `ORDER BY usage_count DESC`.
- 자동완성 시 hot tag 우선.

---

## 4. 인기 태그 / 자동완성

```sql
-- 인기 태그
SELECT * FROM tags ORDER BY usage_count DESC LIMIT 20;

-- 자동완성 (trigram)
SELECT * FROM tags WHERE name ILIKE '맛%' ORDER BY usage_count DESC LIMIT 10;
-- OR (pg_trgm 사용 시)
SELECT * FROM tags WHERE name % '맛집' ORDER BY similarity(name, '맛집') DESC LIMIT 10;
```

### 4.1 왜 pg_trgm

- ILIKE `'맛%'` = leading wildcard X → 활용. 단 `'%맛%'` = 풀스캔.
- trigram = `'%맛%'` 도 인덱스 활용.
- 한국어도 OK (3 char ngram).

---

## 5. 컬럼 "왜"

### 5.1 categories.UNIQUE (board_id, code)
- 같은 board 에 같은 code 두 category 차단.

### 5.2 categories.sort_order
- 사용자 화면의 표시 순서.

### 5.3 categories.use_yn
- 폐기된 category 도 row 보존 (옛 post 의 category_id 보존).

### 5.4 tags.name UNIQUE
- 같은 이름 두 tag 차단.

### 5.5 tags.usage_count
- 인기 태그 정렬 + 자동완성.
- batch 갱신 (1h) — 매 INSERT 마다 X.

### 5.6 post_categories / post_tags 의 PK 가 composite
- 같은 post 의 같은 category/tag 두 번 X.

### 5.7 ON DELETE CASCADE (post)
- post hard delete 시 매핑 row 자동 삭제.

---

## 6. 함정

### 함정 1 — tag 가 case-sensitive
"맛집" vs "맛집" 다른 row.
→ 저장 전 normalize (lowercase).

### 함정 2 — tag 무한 생성
abuse — 10만 tag.
→ post 당 max 10 tag + 사용자 별 rate limit.

### 함정 3 — pg_trgm 없이 자동완성
풀스캔.
→ trigram extension.

### 함정 4 — usage_count 매 INSERT 갱신
hot tag UPDATE 폭주.
→ 1h batch.

### 함정 5 — category 폐기 시 옛 post 영향
post.category_id 가 dangling.
→ use_yn='N' (row 보존).

### 함정 6 — post 삭제 시 매핑 안 정리
좀비 row.
→ ON DELETE CASCADE.

### 함정 7 — tag UNIQUE 누락
중복 가능.
→ name UNIQUE.

---

## 7. 관련

- [[database|↑ hub]]
- [[posts-table]] — post 의 category / tag 참조
- [[../implementation/taxonomy-impl]] (todo)
