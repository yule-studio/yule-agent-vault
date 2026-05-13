---
title: "GraphQL — Schema-based Query"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:10:00+09:00
tags:
  - network
  - graphql
  - api
---

# GraphQL — Schema-based Query

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Schema / Query / Mutation / Subscription |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **시작** | Facebook (2012 내부, 2015 오픈) |
| **표준** | GraphQL Foundation (Linux) |
| **전송** | HTTP POST (보통) / WebSocket |
| **포맷** | JSON |
| **IDL** | GraphQL SDL |

---

## 1. 한 줄 정의

**클라이언트가 원하는 데이터만 쿼리** — 한 endpoint, 자유 쿼리 언어, 강한 타입. REST 의 over-fetch / under-fetch 해결.

---

## 2. 왜 GraphQL?

### REST 의 한계
- **Over-fetch** — 필요 없는 필드도 받음
- **Under-fetch** — 여러 endpoint 호출 (N+1)
- 버전 관리 — v1, v2
- 클라이언트 마다 다른 요구

### GraphQL 해결
- 한 쿼리 — 필요한 만큼만
- 한 endpoint — `/graphql`
- 스키마 진화 — deprecation
- Strongly typed

---

## 3. Schema — SDL

```graphql
type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
}

type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
}

type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
}

type Query {
    user(id: ID!): User
    users: [User!]!
    post(id: ID!): Post
}

type Mutation {
    createUser(name: String!, email: String!): User!
    updatePost(id: ID!, title: String, content: String): Post!
    deletePost(id: ID!): Boolean!
}

type Subscription {
    postAdded: Post!
    commentAdded(postId: ID!): Comment!
}
```

### Type
- Scalar — Int, Float, String, Boolean, ID
- Object — type User
- Enum / Union / Interface
- Input — 변형 인자
- ! — non-null
- [] — list

---

## 4. Query — 쿼리

### 요청
```graphql
query {
    user(id: "1") {
        name
        email
        posts {
            title
            comments {
                text
            }
        }
    }
}
```

### 응답
```json
{
    "data": {
        "user": {
            "name": "Alice",
            "email": "alice@a.com",
            "posts": [
                {
                    "title": "Hello",
                    "comments": [
                        {"text": "Nice"}
                    ]
                }
            ]
        }
    }
}
```

→ 한 호출 — 필요한 정보만.

---

## 5. Mutation — 변경

```graphql
mutation {
    createUser(name: "Alice", email: "alice@a.com") {
        id
        name
    }
}
```

### 응답 형식
- Mutation 도 응답 데이터 반환
- 변경 후 즉시 새 상태

---

## 6. Subscription — 실시간

```graphql
subscription {
    postAdded {
        id
        title
        author {
            name
        }
    }
}
```

### 전송
- WebSocket (graphql-ws / graphql-transport-ws)
- SSE 도 가능

### 사용
- 실시간 알림 / 채팅 / 라이브 데이터

---

## 7. Variables / Operations

```graphql
query GetUser($id: ID!) {
    user(id: $id) {
        name
    }
}

# 변수
{
    "id": "1"
}
```

### 효과
- 쿼리 캐시 가능 (같은 op 다른 변수)
- 보안 (SQL injection 방지)

---

## 8. Resolver

### 정의
- 각 field 의 데이터 fetch 함수

### Apollo (Node.js)
```javascript
const resolvers = {
    Query: {
        user: (parent, { id }, context) => db.users.findById(id),
        users: () => db.users.findAll(),
    },
    User: {
        posts: (user) => db.posts.findByAuthor(user.id),
    },
    Post: {
        author: (post) => db.users.findById(post.authorId),
        comments: (post) => db.comments.findByPost(post.id),
    },
};
```

### 단계별 호출
```
query user → 1 호출
  → User.posts → 1 호출 (user 별)
    → Post.comments → N 호출 (post 별)    ← N+1 문제
```

---

## 9. N+1 — DataLoader

### 문제
```
posts 100 개 → comments 100 query (N+1)
```

### DataLoader
```javascript
const commentLoader = new DataLoader(async (postIds) => {
    return await db.comments.findByPostIds(postIds);
});

// Resolver
comments: (post, args, { loaders }) => loaders.comment.load(post.id)
```

→ 같은 tick 의 모든 호출 — 한 batch query.

### 효과
- 100 query → 1 query
- 캐시 (request-scope)

---

## 10. 인증 / 권한

### Context
```javascript
const server = new ApolloServer({
    typeDefs,
    resolvers,
    context: ({ req }) => {
        const token = req.headers.authorization;
        const user = verifyJWT(token);
        return { user };
    }
});

// Resolver
user: (parent, args, context) => {
    if (!context.user) throw new AuthenticationError();
    return db.users.findById(args.id);
}
```

### Directive
```graphql
directive @auth(role: Role) on FIELD_DEFINITION

type Query {
    secret: String @auth(role: ADMIN)
}
```

---

## 11. GraphQL vs REST vs gRPC

| | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| **Endpoint** | 여러 | 한 `/graphql` | 여러 (service.method) |
| **포맷** | JSON | JSON | Protobuf |
| **스키마** | OpenAPI | SDL | proto |
| **Fetch** | 여러 호출 | 한 쿼리 | 한 호출 (보통) |
| **Streaming** | SSE / WS | Subscription | gRPC streaming |
| **버전** | URL | deprecation | field number |
| **캐시** | 좋음 (HTTP cache) | 어려움 | 약함 |
| **Browser** | 좋음 | 좋음 | gRPC-Web 필요 |
| **사용** | Public REST | 클라이언트 친화 | 마이크로서비스 |

---

## 12. 도구 / 생태계

### 서버
- **Apollo Server** (Node.js, 가장 흔함)
- **GraphQL Yoga** (Node.js)
- **Hasura** — DB → GraphQL 자동
- **PostGraphile** — PostgreSQL → GraphQL
- **Strawberry** (Python)
- **graphql-go** (Go)
- **gqlgen** (Go, codegen)

### 클라이언트
- **Apollo Client** (React / iOS / Android)
- **Relay** (Facebook)
- **urql** (lighter Apollo)
- **graphql-request** (단순)

### 개발 도구
- **GraphiQL / GraphQL Playground**
- **Apollo Studio** — schema registry / metrics
- **GraphQL Inspector** — breaking change detect

---

## 13. Federation — 분산 GraphQL

### 문제
- 큰 회사 — 여러 팀 / 여러 GraphQL
- 단일 schema 어떻게 합치나?

### Apollo Federation
- Subgraph (각 팀) + Gateway (합침)
- @key, @extends 디렉티브
- Gateway 가 쿼리 분배 / 결합

```graphql
# Users subgraph
type User @key(fields: "id") {
    id: ID!
    name: String!
}

# Posts subgraph
extend type User @key(fields: "id") {
    id: ID! @external
    posts: [Post!]!
}
```

→ 클라이언트 — 한 endpoint, 내부 federation.

---

## 14. 보안 / 함정

### 함정 1 — Query depth attack
```graphql
{ user { friends { friends { friends { ... } } } } }
```

→ DoS. depth limit / cost analysis.

### 함정 2 — Query complexity
- 노드 가중치 — 한계 점수 (Cost analysis)
- graphql-cost-analysis

### 함정 3 — N+1 안 처리
DataLoader 필수.

### 함정 4 — Public schema introspection
운영 — introspection 비활성. APQ (Automatic Persisted Queries).

### 함정 5 — Caching 어려움
한 endpoint POST — HTTP cache X.
Persisted query → GET URL → cache.

### 함정 6 — Error 처리의 부분 성공
```json
{
    "data": { "user": {...}, "post": null },
    "errors": [{"message": "Post not found"}]
}
```

→ 한 쿼리 일부 실패 — 클라이언트 처리.

### 함정 7 — Schema 의 breaking change
Field 제거 / 타입 변경 — 클라이언트 깨짐.
@deprecated / 천천히.

---

## 15. 학습 자료

- graphql.org (공식)
- "Apollo GraphQL" docs
- "Production Ready GraphQL" (Marc-André Giroux)
- "The Fullstack Tutorial for GraphQL" (Prisma)

---

## 16. 관련

- [[rpc-messaging]] — Hub
- [[grpc]] — 비교
- [[websocket]] — Subscription
- [[../http/methods/methods]] — REST 비교
