---
title: "user_blocks 테이블 — 차단 사용자"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - blocks
---

# user_blocks 테이블 — 차단 사용자

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 양방향 차단. 정책: [[../design-decisions/block-policy]].

---

## 1. Schema

```sql
-- V13__create_user_blocks.sql
CREATE TABLE user_blocks (
    blocker_id  CHAR(26) NOT NULL,
    blocked_id  CHAR(26) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id),
    CONSTRAINT chk_no_self_block CHECK (blocker_id != blocked_id)
);

CREATE INDEX ix_user_blocks_blocker ON user_blocks (blocker_id);
CREATE INDEX ix_user_blocks_blocked ON user_blocks (blocked_id);
```

---

## 2. 컬럼 "왜"

### 2.1 PK (blocker, blocked)
- 같은 차단 두 번 X.
- 사용자가 차단했는지 O(1) lookup.

### 2.2 self-block CHECK
- A 가 A 차단 = 자기 글 안 보임 (재앙).

### 2.3 created_at
- "언제 차단했는지" audit.
- "최근 차단 list" UI.

---

## 3. 조회 — 양방향 block list

```sql
-- viewer 와 관련된 모든 차단 (양방향)
SELECT DISTINCT user_id FROM (
    SELECT blocked_id AS user_id FROM user_blocks WHERE blocker_id = :viewerId
    UNION
    SELECT blocker_id AS user_id FROM user_blocks WHERE blocked_id = :viewerId
) blocks;
```

→ Redis cache 1h ([[../design-decisions/block-policy#4.1]]).

---

## 4. 한도

```yaml
max-blocks-per-user: 1000
```

```java
if (userBlocks.countByBlockerId(userId) >= 1000) {
    throw new BusinessException(ResponseCode.LIMIT_EXCEEDED,
        "차단 사용자 한도 (1000) 초과");
}
```

---

## 5. 함정

### 함정 1 — Self-block 허용
모든 자기 글 안 보임.
→ CHECK 제약.

### 함정 2 — Block 한도 없음
무한 차단 = 게시판 비활성화.
→ 1000 한도.

### 함정 3 — Hard delete 시 좀비
user delete 시 user_blocks 의 dangling FK.
→ 옵션 FK + CASCADE.

### 함정 4 — Block list 가 너무 크면 query 부담
1000 user 의 block 매 query JOIN.
→ Redis cache + application filter (in-memory).

---

## 6. 관련

- [[database|↑ hub]]
- [[../design-decisions/block-policy]]
- [[../implementation/moderation-impl]] (todo)
