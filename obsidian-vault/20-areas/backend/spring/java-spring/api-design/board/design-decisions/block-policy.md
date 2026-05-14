---
title: "차단 사용자 정책 — block list + filter"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - design-decisions
  - block
---

# 차단 사용자 정책 — block list + filter

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> "사용자가 다른 user 차단" — 개인 차단 (모더레이션과 다름).

---

## 1. 본 vault 결정

- **양방향 차단** — A 가 B 차단 시, 둘 다 서로 글 / 댓글 안 보임.
- **즉시 적용** — Redis cache + DB.
- **조회 시 filter** — 모든 list query 에 block list join.

---

## 2. 옵션 비교

### 2.1 단방향 차단

```
A 가 B 차단 시:
- A 의 화면에서 B 의 글 사라짐
- B 는 A 의 글 그대로 봄 (B 는 자기가 차단됐는지 모름)
```

**왜 적합**
- 사생활 보호 — A 가 B 모르게 차단.
- B 의 어뷰즈 행동 변화 X (자기가 차단된 줄 모름).

**왜 안 됨**
- B 가 A 의 글에 댓글 / 신고 → A 가 알 수 있음 (의도된 격리 깨짐).

---

### 2.2 양방향 차단 (본 vault)

```
A 가 B 차단 시:
- A 의 화면에서 B 안 보임
- B 의 화면에서 A 안 보임 (B 는 차단됐는지 모름)
```

**왜 적합**
- 완전 격리.
- 한국 SaaS (당근 / 무신사) 표준.

**구현**
```java
// A 가 B 차단 INSERT
// 조회 시: WHERE author_id NOT IN (block list of viewer)
//          AND viewer_id NOT IN (block list of author)
```

---

## 3. DB 스키마

```sql
CREATE TABLE user_blocks (
    blocker_id  CHAR(26) NOT NULL REFERENCES users(id),
    blocked_id  CHAR(26) NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id),
    CONSTRAINT chk_no_self_block CHECK (blocker_id != blocked_id)
);

CREATE INDEX ix_user_blocks_blocker ON user_blocks (blocker_id);
CREATE INDEX ix_user_blocks_blocked ON user_blocks (blocked_id);
```

### 3.1 왜 PK (blocker, blocked)

- 같은 차단 두 번 X.
- 차단 조회 O(1).

### 3.2 왜 self-block CHECK

- A 가 A 차단 = 의미 X + 모든 자기 글 안 보임 (재앙).

---

## 4. 조회 시 Filter

```sql
-- viewer = current user
-- author_id 가 viewer 의 차단 list 에 있으면 제외
-- viewer_id 가 author 의 차단 list 에 있으면 제외 (양방향)

SELECT * FROM posts p
WHERE p.status = 'PUBLISHED'
  AND p.author_id NOT IN (
      SELECT blocked_id FROM user_blocks WHERE blocker_id = :viewerId
  )
  AND p.author_id NOT IN (
      SELECT blocker_id FROM user_blocks WHERE blocked_id = :viewerId
  )
ORDER BY ...
```

**또는 — UNION 단일 query**:

```sql
WITH blocked AS (
    SELECT blocked_id AS uid FROM user_blocks WHERE blocker_id = :viewerId
    UNION
    SELECT blocker_id AS uid FROM user_blocks WHERE blocked_id = :viewerId
)
SELECT * FROM posts
WHERE status = 'PUBLISHED' AND author_id NOT IN (SELECT uid FROM blocked)
ORDER BY ...
```

### 4.1 Redis cache (조회 빈도 ↑)

```
Key:   user:blocks:{userId}
Value: SET of blocked_id + blocker_id (양방향 모두)
TTL:   1h
```

```java
public Set<UserId> getBlockedUsers(UserId viewerId) {
    String key = "user:blocks:" + viewerId;
    Set<String> cached = redis.opsForSet().members(key);
    if (cached != null && !cached.isEmpty()) {
        return cached.stream().map(UserId::new).collect(Collectors.toSet());
    }

    // DB miss → load + cache
    var fromDb = userBlocks.findAllRelatedTo(viewerId);
    redis.opsForSet().add(key, fromDb.stream().map(UserId::value).toArray(String[]::new));
    redis.expire(key, Duration.ofHours(1));
    return fromDb;
}
```

### 4.2 application 단 filter

```java
var posts = postRepo.findRecentByBoard(boardId, cursor, limit);
var blocked = blockService.getBlockedUsers(viewerId);
return posts.stream()
    .filter(p -> !blocked.contains(p.authorId()))
    .toList();
```

→ DB query 부담 ↓ (block list 자체가 보통 작음 — 10-50).

**왜 application filter**
- block list 가 크지 않음 (개인 차단 보통 < 100).
- DB JOIN 부담 ↓.
- 단 — 페이지 boundary 영향 (filter 후 18 row).

자세히: [[../implementation/moderation-impl#5 차단]].

---

## 5. 차단 endpoint

```http
POST /api/v1/users/{userId}/block
Authorization: Bearer <access>

200 OK
{ "blockedAt": "2026-05-15T11:00:00Z" }
```

```http
DELETE /api/v1/users/{userId}/block
(unblock)
```

```http
GET /api/v1/me/blocks
(차단 list)
```

---

## 6. 차단 list 한도

```yaml
max-blocks-per-user: 1000
```

**왜 한도**
- 무한 차단 = abuse (게시판 전체 비활성화).
- 1000 = 일반 사용자 충분.

---

## 7. 함정 모음

### 함정 1 — Block filter 없음
차단한 user 의 글 / 댓글 계속 봄.
→ 모든 list query 에 filter.

### 함정 2 — 단방향 차단만
B 가 A 의 글에 댓글 → A 가 봄.
→ 양방향.

### 함정 3 — Block cache 없음
매 query 마다 DB JOIN — 부담.
→ Redis 1h cache.

### 함정 4 — Self-block 허용
A 가 A 차단 → 모든 자기 글 안 보임.
→ DB CHECK.

### 함정 5 — 차단 후 옛 댓글 / 좋아요 row 그대로
차단 의미 약화 (옛 데이터 그대로).
→ 정책 결정 — 새 콘텐츠만 filter 하는 게 일반.

### 함정 6 — admin 도 block 영향
admin 이 신고 글 검토 시 차단된 사용자 글 안 보임.
→ admin role 은 filter bypass.

### 함정 7 — Block list 무제한
무한 차단 = abuse.
→ 1000 한도.

### 함정 8 — DM 의 block 미반영
차단 후 DM 받음.
→ DM 도 block filter (별도 chat 시스템).

---

## 8. 다른 컨텍스트

### 8.1 익명 게시판 (디시)

```yaml
block: IP-based (user-id 없음)
implementation: IP 차단 list
```

### 8.2 실명 (네이버 카페)

```yaml
block: per-user
moderation: 강제 실명 정책으로 보완
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[moderation-policy]] — 글로벌 모더 (개인 차단과 다름)
- [[../implementation/moderation-impl]]
- [[../database/database]] — user_blocks 테이블
