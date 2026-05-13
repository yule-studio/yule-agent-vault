---
title: "REST (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:35:00+09:00
tags:
  - network
  - http
  - rest
  - api
---

# REST (Hub) — Representational State Transfer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | REST 6 제약 + 자원 설계 + HATEOAS |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**Roy Fielding 의 박사논문 (2000)** — HTTP 위 분산 시스템 아키텍처 스타일. 6 제약을
만족하면 RESTful.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2000 | Roy Fielding — "Architectural Styles" (UC Irvine PhD) |
| 2005+ | Web 2.0 — REST API 폭발 |
| 2007 | Richardson Maturity Model |
| 2013+ | OpenAPI (Swagger 2.0) |
| 2017 | GraphQL — REST 의 대안 (Facebook) |
| 2024+ | tRPC, gRPC-Web 등 — REST 와 공존 |

---

## 3. REST 의 6 제약 (Fielding 2000)

### 3.1 Client-Server
- 분리 — 독립 진화
- UI 와 데이터 분리

### 3.2 Stateless
- 서버가 클라 상태 X
- 매 요청 자기 정보
- → JWT, Session ID (cookie 의 sessionid 만 — 응용 입장 stateless)

### 3.3 Cacheable
- 응답 캐시 가능 / 불가 명시
- HTTP `Cache-Control`, `ETag` 활용

### 3.4 Layered System
- 클라가 직접 vs proxy 구분 X
- LB / CDN / Gateway 투명하게 끼움

### 3.5 Uniform Interface (가장 중요)
- 자원 식별 (URI)
- 표현 (JSON / XML / ...)
- Self-descriptive messages (HTTP method + headers + body)
- HATEOAS — Hypermedia as State Engine

### 3.6 Code on Demand (옵션)
- 서버가 클라에 코드 전달 (JS) — 옵션 제약

자세히 → [[rest-principles]]

---

## 4. 자원 (Resource) 중심

```
GET /users           ← 사용자 목록
GET /users/123       ← 사용자 1 명
POST /users          ← 새 사용자 생성
PUT /users/123       ← 갱신
DELETE /users/123    ← 삭제
GET /users/123/orders ← 사용자 의 주문들
```

→ URI = **명사** (자원), HTTP 메서드 = **동사** (행동).

### 안티패턴 — RPC 스타일
```
POST /getUserById
POST /deleteUser
POST /createOrder
```

→ 동사가 URI 에 — RPC, REST X.

자세히 → [[resource-design]]

---

## 5. Richardson Maturity Model

### Level 0 — RPC over HTTP
- 한 URL + POST 만
- SOAP / XML-RPC

### Level 1 — Resources
- URI 가 자원 명사
- 메서드는 POST 만 (또는 모름)

### Level 2 — HTTP Verbs
- GET / POST / PUT / DELETE 의미 활용
- 상태 코드 활용 (201, 404 등)
- **현실의 "REST"** 대부분

### Level 3 — Hypermedia (HATEOAS)
- 응답에 다음 가능한 action 의 link
- 진정한 REST

자세히 → [[hateoas]]

---

## 6. 표준 응답 형식 예

### REST API 응답 (JSON)

```http
HTTP/1.1 200 OK
Content-Type: application/json
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "created_at": "2026-05-13T10:00:00Z"
}
```

### Collection

```http
GET /users?page=1&limit=20 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
X-Total-Count: 1000

{
  "data": [{...}, {...}],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1000
  },
  "links": {
    "next": "/users?page=2&limit=20",
    "prev": null
  }
}
```

### Error (RFC 7807)

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://example.com/probs/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email is invalid",
  "instance": "/users",
  "errors": [
    {"field": "email", "code": "invalid_format"}
  ]
}
```

---

## 7. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[rest-principles]] | 6 제약 + Stateless / Cacheable / Uniform Interface |
| [[resource-design]] | URI / Naming / Nested resource / Singleton |
| [[hateoas]] | Hypermedia / Maturity Model Level 3 |
| [[api-versioning]] | URL / Header / Media type versioning |
| [[rest-vs-rpc-graphql]] | 비교 |

---

## 8. HTTP 메서드 ↔ CRUD

| HTTP | CRUD | URL 예 |
| --- | --- | --- |
| GET | Read | `/users/123` |
| POST | Create | `/users` |
| PUT | Update (전체) | `/users/123` |
| PATCH | Update (부분) | `/users/123` |
| DELETE | Delete | `/users/123` |

자세히 → [[../methods/methods]]

---

## 9. 표준 응답 코드

| 코드 | 의미 |
| --- | --- |
| 200 OK | 성공 (body 있음) |
| 201 Created | POST 로 생성 + Location |
| 202 Accepted | 비동기 시작 |
| 204 No Content | DELETE / PUT 성공 (body X) |
| 400 Bad Request | 잘못된 형식 |
| 401 Unauthorized | 인증 |
| 403 Forbidden | 인가 |
| 404 Not Found | 자원 없음 |
| 409 Conflict | 충돌 |
| 422 Unprocessable | 검증 실패 |
| 429 Too Many Requests | Rate limit |
| 500 / 502 / 503 | 서버 오류 |

자세히 → [[../status-codes/status-codes]]

---

## 10. 실제 REST API 의 함정

### 10.1 진정한 REST 가 아님
- 대부분 Level 2 (Maturity)
- HATEOAS 거의 안 함
- 그래도 "REST API" 라 부름

### 10.2 RPC 스타일이 섞임
```
POST /users/123/activate
POST /api/jobs/start
```

- 자원 모델로 표현 어려운 액션
- 실용 — RESTful 보다 명확하면 OK

### 10.3 일관성 부족
- 같은 응용에 다른 패턴
- 페이지네이션 / 에러 / 인증 패턴

---

## 11. JSON:API / HAL / Hydra

표준화 시도:

### JSON:API
```json
{
  "data": {
    "type": "users",
    "id": "123",
    "attributes": {...},
    "relationships": {...}
  },
  "links": {...}
}
```

### HAL (Hypertext Application Language)
```json
{
  "_embedded": {...},
  "_links": {
    "self": {"href": "/users/123"},
    "orders": {"href": "/users/123/orders"}
  }
}
```

### OpenAPI / Swagger
- 명세 표준 — REST API 문서화
- Codegen / Mock 가능

---

## 12. 모던 대안

### GraphQL
- 한 endpoint + 클라가 query 명세
- Over/Under-fetching 해결
- 자세히 → [[../../rpc-messaging/rpc-messaging]]

### gRPC
- HTTP/2 + Protobuf
- 빠름 + 강 타입
- Microservice 내부

### tRPC
- TypeScript end-to-end 타입
- 단일 stack (TS) 환경

### REST 의 자리
- 공개 API (외부 개발자)
- 표준 / 호환성 중요
- 모바일 / 웹

---

## 13. 면접 / 토픽

1. **REST 의 6 제약**.
2. **REST vs RPC vs GraphQL**.
3. **HATEOAS** — 의미와 현실.
4. **Stateless 의 의미**.
5. **URI 설계 best practices**.

---

## 14. 학습 자료

- **Roy Fielding's Dissertation** — "Architectural Styles..." (2000)
- **RESTful Web APIs** (Richardson / Amundsen)
- **REST in Practice** (Webber et al.)
- "REST API Tutorial" — restapitutorial.com

---

## 15. 관련

- [[../http]] — HTTP hub
- [[rest-principles]]
- [[resource-design]]
- [[hateoas]]
- [[api-versioning]]
- [[rest-vs-rpc-graphql]]
- [[../methods/methods]]
- [[../status-codes/status-codes]]
