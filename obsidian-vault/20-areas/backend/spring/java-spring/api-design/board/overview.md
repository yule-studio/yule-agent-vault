---
title: "board §1 — 전체 흐름 overview"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - overview
---

# board §1 — 전체 흐름 overview

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 — 커뮤니티 게시판 end-to-end 흐름 |

**[[board|↑ hub]]**  ·  → [[prerequisites]]

> 게시글 작성부터 조회 / 댓글 / 신고까지의 **end-to-end 흐름**.

---

## 1. 사용자 여정 (User Journey)

```mermaid
flowchart TD
    Visit[사용자가 게시판 진입] --> List[게시글 목록]
    List -->|cursor pagination| Scroll[무한 스크롤]
    List -->|클릭| Detail[게시글 상세]
    Detail --> View["view count +1<br/>(Redis counter)"]
    Detail --> Comments[댓글 목록]

    Detail -->|좋아요| Like[Like toggle]
    Detail -->|북마크| Bookmark[Bookmark toggle]
    Detail -->|신고| Report[Report submit]

    Comments -->|작성| NewComment[댓글 작성]
    NewComment -->|대댓글| Reply[대댓글]

    List -->|글쓰기 (인증 필수)| Editor[에디터]
    Editor -->|첨부| Upload[S3 presigned URL]
    Editor -->|발행| Publish[Post create + Outbox]
    Publish -->|알림| Notification[작성자 follower 알림]

    style Detail fill:#fef3c7
    style Editor fill:#dbeafe
    style Report fill:#fecaca
```

---

## 2. 게시글 작성 흐름 (첨부 포함)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant C as 클라
    participant API as Backend
    participant S3
    participant DB
    participant Outbox

    U->>C: 글쓰기 클릭
    C->>API: POST /posts/attachments/presigned { fileName, fileSize }
    API->>API: 파일 크기 / type 검증
    API->>S3: presigned URL 발급 (5분 TTL)
    API-->>C: { uploadUrl, fileKey }

    C->>S3: PUT (직접 업로드 — backend 우회)
    S3-->>C: 200 OK

    C->>API: POST /boards/{boardId}/posts { title, content, attachmentKeys: [...] }
    API->>API: XSS sanitize (content) + 작성자 검증
    rect rgb(254, 243, 199)
    note over API,DB: @Transactional
    API->>DB: posts INSERT (status=PUBLISHED)
    API->>DB: post_attachments INSERT (key 매핑)
    API->>DB: post_tags INSERT (tag 매핑)
    API->>DB: publishEvent(PostCreated)
    end

    rect rgb(219, 234, 254)
    note over Outbox: AFTER_COMMIT
    Outbox->>DB: notification_outbox INSERT (follower 알림)
    end

    API-->>C: 201 Created { postId, url }
```

---

## 3. 댓글 / 대댓글 흐름

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant API
    participant DB
    participant Cache as Redis

    U->>API: POST /posts/{postId}/comments { content }
    API->>API: XSS sanitize + 작성자 검증
    rect rgb(254, 243, 199)
    note over API,DB: @Transactional
    API->>DB: comments INSERT (parent_id=NULL)
    API->>DB: posts UPDATE SET comment_count = comment_count + 1
    end
    API-->>U: 201 Created

    U->>API: POST /comments/{commentId}/reply { content }
    API->>DB: comments INSERT (parent_id={commentId})
    API->>DB: comment_count++ + parent.reply_count++

    Note over API,Cache: 조회는 hot post 의 댓글 cache
    U->>API: GET /posts/{postId}/comments
    API->>Cache: GET comments:{postId}
    alt cache miss
        API->>DB: SELECT comments + tree build (parent_id 기반)
        API->>Cache: SET comments:{postId} TTL 5m
    end
    API-->>U: 댓글 tree
```

---

## 4. 좋아요 / 북마크 — counter 처리

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant Cache as Redis
    participant DB
    participant Batch as Batch Job

    U->>API: POST /posts/{postId}/like
    API->>Cache: HINCRBY post:likes {postId} 1
    API->>DB: post_likes INSERT (user_id, post_id)
    API-->>U: 200 OK

    note over Batch: 매 1시간
    Batch->>Cache: SCAN post:likes:*
    Batch->>DB: posts UPDATE SET like_count = ?

    note over Cache,DB: Redis 가 진실 (eventual sync)<br/>장애 시 SUM(post_likes) 로 복구
```

자세히: [[design-decisions/like-counter]] (todo).

---

## 5. 검색 흐름

```mermaid
flowchart LR
    Q[사용자 검색 query] --> P{posts 수}
    P -->|< 100만| DB[DB ILIKE<br/>posts.title / content]
    P -->|100만~1억| FTS["PostgreSQL FTS<br/>tsvector + GIN index"]
    P -->|1억+| ES["Elasticsearch<br/>별도 인프라"]

    DB --> Result[검색 결과]
    FTS --> Result
    ES --> Result

    style DB fill:#d1fae5
    style FTS fill:#fef3c7
    style ES fill:#fecaca
```

자세히: [[design-decisions/search-strategy]] (todo).

---

## 6. 신고 / 모더레이션

```mermaid
flowchart TD
    U[사용자 신고] --> R[Report INSERT]
    R --> Count{같은 target 신고 수}
    Count -->|< 5| Pending[admin review 대기]
    Count -->|>= 5| Auto[자동 HIDDEN]
    Auto --> Notify[작성자 알림]
    Pending --> Admin[Admin 대시보드]
    Admin -->|승인| Hidden[Post HIDDEN]
    Admin -->|기각| Reject[Report DISMISSED]
    Hidden --> AuthorNotify[작성자 알림]

    style Auto fill:#fecaca
    style Hidden fill:#fed7aa
```

자세히: [[implementation/moderation-impl]] (todo).

---

## 7. 어떤 endpoint 가 비인증인가

| Endpoint | 인증 | 비고 |
| --- | --- | --- |
| `GET /boards/{id}/posts` | ❌ | 공개 게시판은 비인증 |
| `GET /posts/{id}` | ❌ | 공개 글 |
| `GET /posts/{id}/comments` | ❌ | 공개 댓글 |
| `POST /boards/{id}/posts` | ✅ | 작성자 인증 |
| `POST /posts/{id}/comments` | ✅ | |
| `POST /posts/{id}/like` | ✅ | |
| `POST /posts/{id}/report` | ✅ | |
| `GET /me/posts` | ✅ | |
| `GET /admin/reports` | ✅ ADMIN | |

→ SecurityConfig 매트릭스: [[security/authentication-authorization]] (todo).

---

## 8. 각 흐름의 핵심 결정점

| 단계 | 결정 | 영향 |
| --- | --- | --- |
| 게시글 — content 형식 | markdown / HTML / JSON (Slate / TipTap) | 에디터 / 렌더링 / XSS |
| 댓글 — depth | flat / 2-level / 무한 | UX / 쿼리 복잡도 |
| 좋아요 counter | DB / Redis / hybrid | 동시성 / latency |
| 정렬 (hot) | 시간 감쇠 algorithm | 메인 화면 노출 |
| 검색 | DB FTS / Elasticsearch | 인프라 / 정확도 |
| 첨부 — 업로드 경로 | 서버 경유 / S3 presigned | 비용 / latency |
| 페이지네이션 | offset / cursor | UX / 성능 |
| 익명 옵션 | 익명 가능 / 닉네임만 / 실명 강제 | 운영 / abuse |
| 신고 자동 hide threshold | 3 / 5 / 10 | false positive vs 빠른 대응 |

각 결정의 trade-off + 권장: [[design-decisions/design-decisions]].

---

## 9. 이 폴더 코드의 재사용성

### A. 도메인 / 표준 부분 (그대로 복사)

- Aggregate (Post / Comment / Like / Report)
- Value Objects (PostId / CommentId / TargetId)
- Repository port + Adapter
- 응답 envelope ([[../../common/response-envelope]])
- Spring Security ([[../../common/security-config]])

→ 거의 모든 커뮤니티 SaaS 에서 그대로 사용 가능.

### B. 정책 / 비즈니스 부분 (조정 필요)

- 게시판 종류 / 분류 (boards 시드)
- 댓글 depth 정책
- 좋아요 / 북마크 활성 여부
- 검색 도구 (DB vs Elasticsearch)
- 첨부 max size / type
- 신고 자동 hide threshold
- 익명 / 닉네임 정책

→ 회사 / 도메인 정책에 따라 조정.

---

## 10. 관련

- [[board|↑ hub]]
- [[prerequisites]] — 다음 (§2)
- [[design-decisions/design-decisions]] — 권장 도구 가이드
- [[../signup/signup]] — 인증 의존성
