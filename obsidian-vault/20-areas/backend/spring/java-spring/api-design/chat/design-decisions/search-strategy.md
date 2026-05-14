---
title: "Search — 메시지 검색"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:18:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, search]
---

# Search — 메시지 검색

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — 단계적

| Phase | 방식 |
| --- | --- |
| F8 | PostgreSQL `ILIKE` (단순) |
| F후 | PostgreSQL FTS (`tsvector` + GIN) |
| F12+ | Elasticsearch (옵션) |

---

## 2. PostgreSQL FTS

```sql
ALTER TABLE messages ADD COLUMN content_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('korean', content)) STORED;

CREATE INDEX ix_msg_tsv ON messages USING GIN (content_tsv);

-- 검색
SELECT id, content, ts_headline('korean', content, query)
FROM messages, plainto_tsquery('korean', '회의') query
WHERE room_id = ? AND content_tsv @@ query
ORDER BY ts_rank(content_tsv, query) DESC
LIMIT 20;
```

---

## 3. 보안 — 본인 room 만

- 모든 검색은 `room_id IN (SELECT room_id FROM room_members WHERE user_id = ?)`.
- → 가입하지 않은 room 의 메시지 검색 X.

---

## 4. 함정

1. **모든 room 검색** → privacy 누설.
2. **한국어 tokenizer** 없음 — `korean` config 또는 별도 plugin (mecab).
3. **partition 별 query 안 됨** → ALL partition scan.
   → `created_at` range 함께.
4. **첨부 파일 검색 X** — 첨부 내용 X (그냥 메시지 text).

---

## 5. 관련

- [[design-decisions|↑ hub]]
- [[../implementation/search-impl]]
- [[../../elasticsearch-integration|↗ ES recipe]]
