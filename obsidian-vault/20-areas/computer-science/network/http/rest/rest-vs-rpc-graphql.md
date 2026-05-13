---
title: "REST vs RPC vs GraphQL"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:59:00+09:00
tags:
  - network
  - http
  - rest
  - graphql
  - rpc
---

# REST vs RPC vs GraphQL

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 4 가지 패러다임 비교 |

**[[rest|↑ REST]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 4 가지 패러다임

| 패러다임 | 사상 | 예 |
| --- | --- | --- |
| **REST** | 자원 + HTTP 메서드 | `GET /users/123` |
| **RPC** | 원격 함수 호출 | `POST /api/getUserById {id: 123}` |
| **GraphQL** | 단일 endpoint + query 언어 | `POST /graphql {query: "..."}` |
| **gRPC** | RPC + Protobuf + HTTP/2 | 빠른 binary |

---

## 2. 비교 표

| 측면 | REST | RPC / gRPC | GraphQL |
| --- | --- | --- | --- |
| **모델** | 자원 (명사) | 함수 (동사) | Query / Mutation |
| **Endpoint** | 다수 (자원별) | 다수 (함수별) | 1 (보통 `/graphql`) |
| **메서드** | HTTP semantics | POST 만 (보통) | POST 만 |
| **Body** | JSON 흔함 | JSON / Protobuf | GraphQL query |
| **Schema** | OpenAPI (옵션) | Protobuf (gRPC) / WSDL (SOAP) | GraphQL Schema 필수 |
| **Type Safety** | 약 (OpenAPI) | 강 | 강 |
| **Over-fetching** | 가능 | 가능 | **해결** |
| **Under-fetching** | 가능 (N+1) | 가능 | **해결** |
| **Caching** | HTTP 기본 | 직접 구현 | 직접 구현 |
| **HTTP 활용** | 강 | 약 (POST 만) | 약 |
| **Real-time** | Polling / SSE | gRPC streaming | Subscription |
| **에러 처리** | HTTP status code | 응답 본문 | 응답 body 의 errors |
| **학습 곡선** | 낮음 | 중 | 높음 |
| **도구** | curl / Postman | gRPC tools | GraphiQL / Apollo |

---

## 3. REST

### 예
```http
GET /users/123
→ {id: 123, name: "Alice", ...}

GET /users/123/orders
→ [{id: "abc"}, ...]
```

### 장점
- HTTP 의 모든 기능 (캐싱 / 상태 코드 / 메서드)
- 직관적
- 공개 API 표준

### 단점
- Over-fetching (필요 X 필드도)
- Under-fetching (여러 요청 필요)
- 많은 endpoint

### 사용
- 공개 REST API
- 단순 CRUD

---

## 4. GraphQL

### Query

```graphql
query {
  user(id: 123) {
    name
    email
    orders {
      id
      total
      items {
        product { name }
      }
    }
  }
}
```

### 응답 (정확히 query 만)

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "...",
      "orders": [
        {"id": "abc", "total": 100, "items": [{"product": {"name": "Book"}}]}
      ]
    }
  }
}
```

### 장점
- **클라가 필요 데이터만** 받음 (over-fetching 해결)
- **한 요청에 여러 자원** (under-fetching 해결)
- 강한 타입 (Schema)
- Subscription (WebSocket)
- 자동 문서 (GraphiQL)

### 단점
- 복잡성 (서버 / 클라 모두)
- HTTP 캐싱 약 (POST + 다른 query)
- N+1 query 가 서버에서 발생 (DataLoader 패턴)
- File upload 어색
- Rate limiting 복잡 (complexity 기반)

### 사용
- 복잡한 클라 (SPA, mobile)
- BFF (Backend for Frontend) — 다양한 클라
- Facebook / GitHub / Shopify

자세히 → [[../../rpc-messaging/rpc-messaging]]

---

## 5. RPC / gRPC

### gRPC 예

```protobuf
// .proto
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc StreamUpdates(UpdateRequest) returns (stream Update);
}

message GetUserRequest { int32 id = 1; }
message User { int32 id = 1; string name = 2; }
```

### 호출
```python
stub = UserServiceStub(channel)
user = stub.GetUser(GetUserRequest(id=123))
```

### 장점
- **매우 빠름** (binary + HTTP/2)
- 강 타입 (Protobuf)
- Streaming (양방향)
- Code generation
- Microservice 내부 표준

### 단점
- 브라우저 직접 X (gRPC-Web 필요)
- 디버깅 어려움 (binary)
- HTTP 캐싱 X
- 공개 API 어려움

### 사용
- Microservice 내부
- High-performance (HFT, 게임 서버)
- Streaming (chat, log streaming)

---

## 6. 결정 트리

```
공개 API + 외부 개발자?
├ Yes → REST 또는 GraphQL
│   ├ 단순 CRUD → REST
│   └ 복잡 (mobile / SPA / BFF) → GraphQL
└ No (내부)
    ├ 빠른 / 강 타입 / streaming → gRPC
    ├ 단순 / 빠른 개발 → REST
    └ 복잡 데이터 그래프 → GraphQL
```

---

## 7. 실제 응용 — 혼합

대부분 회사 — 여러 패러다임 동시:

### Netflix
- 공개 — REST
- 내부 — gRPC + Falcor (자체 GraphQL 비슷)

### GitHub
- v3 — REST
- v4 — GraphQL
- 내부 — gRPC + Protobuf

### Shopify
- 공개 — REST + GraphQL (Admin / Storefront)
- 내부 — gRPC

### Discord
- 공개 — REST
- Voice / Real-time — WebSocket
- 내부 — RPC (Rust)

---

## 8. 캐싱 비교

### REST
```http
GET /users/123
Cache-Control: max-age=300
ETag: "v1"
```
→ HTTP 의 모든 캐싱 (CDN / Browser).

### GraphQL
- POST + 다른 query → 다른 응답
- HTTP 캐싱 어려움
- 해결:
  - GET 변환 (`@apollo/persisted-queries`)
  - 응용 레벨 캐시 (Apollo Cache, Relay Store)

### gRPC
- 응용 레벨 — Memcached / Redis
- HTTP 캐싱 X

---

## 9. 에러 처리

### REST
```http
HTTP/1.1 404 Not Found
{"error": "User not found"}
```

→ HTTP 상태 코드 + body.

### GraphQL
```http
HTTP/1.1 200 OK     ← 항상 200 (보통)
{
  "data": null,
  "errors": [{"message": "User not found", "extensions": {"code": "NOT_FOUND"}}]
}
```

→ 응답 body 안 errors. HTTP 상태 활용 X (논쟁).

### gRPC
- gRPC Status Code (0=OK, 5=NOT_FOUND, ...)
- HTTP/2 trailer 로 전달

---

## 10. Streaming / Real-time

### REST
- Polling (`GET /events?since=...`)
- Long-polling
- SSE (`text/event-stream`)
- WebSocket (별도)

### GraphQL
- Subscription (WebSocket)
```graphql
subscription { newMessage(roomId: 123) { text } }
```

### gRPC
- Server / Client / Bidirectional streaming
- HTTP/2 의 멀티 stream 활용

---

## 11. 함정

### 함정 1 — REST 가 항상 정답 가정
GraphQL / gRPC 가 더 맞는 경우 있음.

### 함정 2 — GraphQL 의 over-engineering
단순 CRUD 에 GraphQL — 비싼 설정.

### 함정 3 — GraphQL 의 N+1
서버 resolver 가 query 마다 DB — DataLoader 필수.

### 함정 4 — gRPC 의 브라우저
gRPC-Web 또는 REST gateway 필요.

### 함정 5 — Schema 동기화
gRPC / GraphQL — schema 변경 시 클라 / 서버 동시 배포 어려움.

### 함정 6 — 혼합 환경의 일관성
REST + GraphQL + gRPC — 응용 코드 복잡. 명확한 경계.

---

## 12. 학습 자료

- "REST vs GraphQL vs gRPC" — Apollo blog
- "Designing GraphQL Schemas" — Production guides
- "gRPC vs REST" — Google
- "Why GraphQL" — Facebook

---

## 13. 관련

- [[rest]] — REST hub
- [[../../rpc-messaging/rpc-messaging]] — gRPC / GraphQL 깊이
- [[../headers/entity-headers]] — Content-Type
- [[../caching/caching]]
