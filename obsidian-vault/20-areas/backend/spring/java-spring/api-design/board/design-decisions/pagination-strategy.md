---
title: "페이지네이션 — offset vs cursor"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - design-decisions
  - pagination
---

# 페이지네이션 — offset vs cursor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> "글 목록 어떻게 페이지" — offset 은 깊은 page 에서 성능 폭망 + 새 글 추가 시 중복.

---

## 1. 본 vault 결정

**Cursor-based** (default — 무한 스크롤) + **offset** (admin 전용).

- 사용자 화면: cursor (`?cursor=...&limit=20`).
- Admin 대시보드: offset (`?page=5&size=20`).

---

## 2. 옵션 비교

### 2.1 Offset (`?page=5&size=20`)

```sql
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 100;     -- page 6
```

**왜 적합 (admin)**
- 특정 page 점프 가능 (page 100).
- 단순.

**왜 안 됨 (사용자)**
- 깊은 page (page 100+) 시 OFFSET 100*20=2000 → DB 가 2000 row scan + skip.
- 새 글 추가 시 page boundary 흔들림 → user 가 같은 글 두 번 봄.

**구체적 문제**
```
시점 1: page 1 fetch (latest 20)
시점 2: 새 글 1개 추가
시점 3: page 2 fetch (offset 20) → 옛 page 1 의 마지막 글이 page 2 의 첫 번째로 (중복)
```

---

### 2.2 Cursor (`?cursor=...&limit=20`)

```sql
SELECT * FROM posts
WHERE (created_at, id) < (?, ?)    -- cursor tuple
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**왜 적합**
- 깊은 page 도 O(log n) — 인덱스 lookup.
- 새 글 추가해도 cursor 의 시점이 고정 — 중복 없음.
- 무한 스크롤 자연.

**왜 안 됨 (admin)**
- 특정 page 점프 불가 (page 100 가려면 1~100 다 거쳐야).

**Cursor 형식**
```json
{
  "cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNS0xNVQxMDowMDowMCIsImlkIjoiMDFIUVhZIn0=",
  "limit": 20
}
```

- base64 (json) — 사용자가 파싱 어려움 (random 같이 보임).
- 안에 `{ createdAt, id }` tuple.

---

## 3. Cursor 구현

```java
@GetMapping("/posts")
public PageResponse<PostResponse> listPosts(
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") @Max(50) int limit
) {
    Cursor parsed = cursor != null ? Cursor.decode(cursor) : Cursor.first();
    var posts = postRepo.findAfter(parsed.createdAt(), parsed.id(), limit + 1);

    boolean hasNext = posts.size() > limit;
    if (hasNext) posts = posts.subList(0, limit);

    String nextCursor = hasNext
        ? Cursor.of(posts.get(posts.size() - 1)).encode()
        : null;

    return new PageResponse<>(posts, nextCursor, hasNext);
}

public record Cursor(Instant createdAt, String id) {
    public String encode() {
        var json = Map.of("createdAt", createdAt.toString(), "id", id);
        return Base64.getUrlEncoder().encodeToString(
            objectMapper.writeValueAsBytes(json));
    }
    public static Cursor decode(String encoded) { /* ... */ }
    public static Cursor first() { return new Cursor(Instant.MAX, "Z"); }
    public static Cursor of(Post p) { return new Cursor(p.createdAt(), p.id().value()); }
}
```

### 3.1 왜 `LIMIT + 1`

- 21 row fetch → 20 표시 + 21번째 존재 = hasNext true.
- 추가 query 없이 hasNext 판단.

### 3.2 왜 base64 encoding

- 사용자에게 cursor 내부 (createdAt + id) 노출 X.
- URL safe.
- 단 — 해석은 가능 (base64 디코드). 진짜 보호 필요시 signed cursor.

---

## 4. Multi-sort cursor

hot score + created_at 정렬 같은 복잡 case:

```sql
ORDER BY hot_score DESC, id DESC

-- cursor = (hot_score, id)
WHERE (hot_score, id) < (?, ?)
ORDER BY hot_score DESC, id DESC
LIMIT 20;
```

**왜 tuple comparison**
- `WHERE hot_score < ? OR (hot_score = ? AND id < ?)` 와 같은 의미.
- 인덱스 활용 ↑.

```java
public record Cursor(double hotScore, String id) {
    // ...
}
```

---

## 5. Index 요구

```sql
-- cursor pagination 핵심 인덱스
CREATE INDEX ix_posts_board_created ON posts (board_id, created_at DESC, id DESC)
    WHERE status = 'PUBLISHED';

CREATE INDEX ix_posts_board_hot ON posts (board_id, hot_score DESC, id DESC)
    WHERE status = 'PUBLISHED';
```

### 5.1 왜 정렬 컬럼 + id 복합

- cursor tuple comparison 의 정확한 매칭.
- 정렬 컬럼만 — 같은 값 (createdAt 같음) 시 cursor 모호.

### 5.2 왜 partial (PUBLISHED)

- HIDDEN / DELETED 제외 → 인덱스 크기 ↓.

---

## 6. Limit 정책

```yaml
default: 20
max: 50
admin-max: 100
```

**왜 max 50**
- 너무 큼 → DB 부담 + 응답 크기 ↑.
- UX (모바일) — 20-30 적당.

---

## 7. 함정 모음

### 함정 1 — Offset 깊은 page
page 100 = OFFSET 2000 → 2000 row scan.
→ cursor.

### 함정 2 — Cursor 가 single column
같은 created_at 의 2 row 시 cursor 모호.
→ (created_at, id) tuple.

### 함정 3 — Cursor base64 안 함
사용자가 cursor 조작 가능 (next page 임의 변경).
→ encoding + 옵션 signed.

### 함정 4 — Index 없이 cursor
WHERE 조건 풀스캔.
→ ORDER BY columns + id 복합 인덱스.

### 함정 5 — Limit 무한
`?limit=10000` → DB 부담 + 메모리.
→ max 50.

### 함정 6 — Cursor 가 다음 sort 변경에 호환
sort=latest 의 cursor 를 sort=hot 에 사용 → 깨짐.
→ cursor 에 sort key 포함.

### 함정 7 — Cursor 가 변경 가능한 컬럼
posts 의 `updated_at` cursor — 사용자가 글 수정 시 cursor 깨짐.
→ created_at 또는 immutable 값.

### 함정 8 — Offset + 새 글 추가 race
중복 / 누락 표시.
→ cursor 로 해결.

---

## 8. 다른 컨텍스트

### 8.1 검색 결과 (relevance 정렬)

```yaml
pagination: cursor (search_score, id)
```

### 8.2 Admin 전용 (특정 page 점프 필요)

```yaml
pagination: offset (page=?&size=?)
```

### 8.3 데이터 분석 (전체 export)

```yaml
pagination: cursor (id ASC) 또는 keyset
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[hot-ranking]] — 정렬 결합
- [[../implementation/search-pagination-impl]]
- [[../../signup/database/users-table#3 인덱스]]
