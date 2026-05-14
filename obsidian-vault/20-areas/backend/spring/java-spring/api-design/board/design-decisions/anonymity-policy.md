---
title: "익명 / 닉네임 정책"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - design-decisions
  - anonymity
---

# 익명 / 닉네임 정책

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> "글쓴이 표시" — 익명 / 닉네임 / 실명. board 별 정책.

---

## 1. 본 vault 결정

**Board 별 정책** — 게시판마다 `display_mode` 컬럼:

- `NICKNAME`: 닉네임 표시 (default — 한국 SaaS 표준)
- `ANONYMOUS`: 익명 (자유게시판 / 고민상담)
- `REAL_NAME`: 실명 강제 (학교 / 회사 internal)

내부 audit 은 항상 user_id 보존 (모더 / abuse 추적 가능).

---

## 2. 옵션 비교

### 2.1 NICKNAME (본 vault default)

- 가입 시 닉네임 설정 (`users.nickname`).
- 글 / 댓글에 닉네임 표시.
- abuse 추적: user_id 로 admin 가능.

**왜 적합**
- 한국 SaaS 표준 (당근 / 무신사 / 디시).
- 사용자 정체성 (반복 user 식별) + 사생활 보호.

### 2.2 ANONYMOUS

- 글마다 random 익명 ID (예: "익명1234").
- 같은 글의 댓글들도 같은 익명 ID (글 안 일관).
- 다른 글 가면 다른 익명 ID.

**구현**
```java
String anonymousId = "익명" + hash(authorId + postId).substring(0, 4);
```

**왜 적합**
- 민감 토론 (고민 / 정치) 의 사생활.

**왜 안 됨 (일반)**
- abuse 추적 어려움 (admin 만 user_id 보임).
- 일관 정체성 X.

### 2.3 REAL_NAME

- 가입 시 본인 인증 (PASS / NICE) 강제.
- 실명 + 가능하면 휴대폰 매핑.

**언제 적합**
- 학교 / 회사 internal board.
- 한국 청소년 게임 board (실명 인증 의무).

**왜 안 됨 (일반 커뮤니티)**
- 사용자 가입 friction ↑.
- 신원 노출 위험.

---

## 3. DB schema

```sql
ALTER TABLE boards ADD COLUMN display_mode VARCHAR(20) NOT NULL DEFAULT 'NICKNAME';
-- CHECK (display_mode IN ('NICKNAME', 'ANONYMOUS', 'REAL_NAME'))

ALTER TABLE users ADD COLUMN nickname VARCHAR(30) UNIQUE;
-- ALTER TABLE users ADD COLUMN real_name VARCHAR(50);  -- REAL_NAME board 사용 시
```

---

## 4. 응답 처리

```java
public PostResponse toResponse(Post post, Board board) {
    var author = userRepo.findById(post.authorId()).orElseThrow();
    String displayName = switch (board.displayMode()) {
        case NICKNAME -> author.nickname();
        case ANONYMOUS -> "익명" + hash(post.authorId().value() + post.id().value()).substring(0, 4);
        case REAL_NAME -> author.realName();
    };
    return new PostResponse(
        post.id().value(),
        displayName,                    // user_id 노출 X (admin 응답에는 포함)
        post.title(),
        post.content(),
        // ...
    );
}
```

### 4.1 왜 user_id 응답에 X (NICKNAME / ANONYMOUS)

- 사용자 식별자 노출 = enumeration / phishing 위험.
- 응답에는 표시명만.
- admin 응답 (admin endpoint) 에는 user_id 포함.

### 4.2 왜 same post 안 익명 ID 일관

- post 안 댓글 흐름 추적 OK ("익명1234 가 다시 댓글 함").
- 다른 post = 다른 익명 ID → cross-post 추적 차단.

---

## 5. 함정 모음

### 함정 1 — Anonymous 인데 user_id 응답에
사실상 실명 노출.
→ display name 만.

### 함정 2 — Anonymous ID 가 user_id 그대로
완전 식별.
→ hash + truncate.

### 함정 3 — Real name 강제하면서 본인 인증 X
가짜 이름 가능.
→ PASS / NICE 필수.

### 함정 4 — admin 도 anonymous
모더레이션 / abuse 추적 X.
→ admin role 은 user_id 응답.

### 함정 5 — 닉네임 unique 안 함
같은 닉네임 사용자 다수.
→ UNIQUE.

### 함정 6 — 닉네임 변경 무제한
identity 추적 어려움 + 사기.
→ 30일 1회 / 사용자 history.

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[../signup/database/users-table]] — nickname / real_name 컬럼
- [[block-policy]] — 차단 (anonymous 환경에서 어려움)
