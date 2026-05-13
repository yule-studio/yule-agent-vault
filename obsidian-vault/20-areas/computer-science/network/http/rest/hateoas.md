---
title: "HATEOAS — Hypermedia as Engine of State"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:50:00+09:00
tags:
  - network
  - http
  - rest
  - hateoas
  - hypermedia
---

# HATEOAS — Hypermedia as Engine of State

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Maturity Level 3 / HAL / JSON:API / 현실 |

**[[rest|↑ REST]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응답에 **다음 가능한 action 의 link** 포함 — 클라가 URL 하드코딩 X. REST 의
**Uniform Interface** 의 4 번째 sub-원칙. Maturity Level 3 (가장 RESTful).

---

## 2. 의미 — 비유

```
HTML 페이지의 동작:
- 사용자가 페이지 보고 <a>, <form> 등으로 다음 action 결정
- URL 외울 필요 X — 링크 클릭

API 의 HATEOAS:
- 클라이언트가 응답 받고 _links 의 다음 action 결정
- URL 하드코딩 X — link 따라감
```

→ 웹 페이지처럼 — 자기 설명 (self-descriptive).

---

## 3. HATEOAS 응답 예

### 주문 상태에 따라 다른 action

```json
{
  "id": "abc",
  "status": "pending",
  "items": [...],
  "total": 100,
  "_links": {
    "self": {"href": "/orders/abc"},
    "pay": {"href": "/orders/abc/pay", "method": "POST"},
    "cancel": {"href": "/orders/abc/cancel", "method": "POST"},
    "items": {"href": "/orders/abc/items"}
  }
}
```

```json
{
  "id": "abc",
  "status": "paid",
  "_links": {
    "self": {"href": "/orders/abc"},
    "refund": {"href": "/orders/abc/refund", "method": "POST"},
    "ship": {"href": "/orders/abc/ship", "method": "POST"},
    "track": {"href": "/orders/abc/tracking"}
  }
}
```

→ 같은 자원 — 상태 별 다른 가능 action.

---

## 4. 장점

### 4.1 클라가 URL 무관
- URL 변경 시 클라 코드 X 변경 (link 따라감)
- 서버가 자유롭게 URL 진화

### 4.2 가능한 action 명시
- 클라가 "지금 무엇 할 수 있나" 학습
- UI 자동 생성 가능 (조건부 버튼)

### 4.3 진정한 REST
- Fielding 의 Maturity Level 3
- Hypermedia driven

---

## 5. 단점 / 현실

### 5.1 복잡성
- 응답 크기 ↑ (links)
- 표준 X (HAL / JSON:API / 자체)

### 5.2 클라가 정말 동적인가
- 대부분 클라가 URL 의 의미 알고 코딩
- "next page" 정도만 link 활용

### 5.3 표준 없음
- HAL / JSON:API / Hydra / 자체 — 통일 X

### 5.4 모바일 / 작은 응답 비효율
- 매 응답에 links — 대역폭

### 5.5 GraphQL / RPC 의 부상
- 명시적 contract (Schema) 가 더 명확
- HATEOAS 의 동적성 가치 ↓

→ **현실: HATEOAS 거의 X**. Level 2 가 보통.

---

## 6. HAL (Hypertext Application Language)

```json
{
  "id": 123,
  "name": "Alice",
  "_embedded": {
    "orders": [
      {"id": "abc", "_links": {"self": {"href": "/orders/abc"}}}
    ]
  },
  "_links": {
    "self": {"href": "/users/123"},
    "orders": {"href": "/users/123/orders"},
    "edit": {"href": "/users/123", "method": "PUT"}
  }
}
```

### 특징
- `_links` — 가능 action
- `_embedded` — 관련 자원 포함
- Content-Type: `application/hal+json`

---

## 7. JSON:API

```json
{
  "data": {
    "type": "users",
    "id": "123",
    "attributes": {
      "name": "Alice",
      "email": "alice@..."
    },
    "relationships": {
      "orders": {
        "links": {"related": "/users/123/orders"},
        "data": [{"type": "orders", "id": "abc"}]
      }
    },
    "links": {"self": "/users/123"}
  },
  "included": [
    {"type": "orders", "id": "abc", "attributes": {...}}
  ]
}
```

### 특징
- 표준 (https://jsonapi.org/)
- relationships + included
- pagination, sparse fieldset, sort, filter 표준화

---

## 8. Hydra

```json
{
  "@context": "http://www.w3.org/ns/hydra/context.jsonld",
  "@id": "/orders/abc",
  "@type": "Order",
  "status": "pending",
  "operation": [
    {
      "@type": "Operation",
      "method": "POST",
      "target": "/orders/abc/pay",
      "expects": "Payment"
    }
  ]
}
```

### 특징
- JSON-LD 기반 (Linked Data)
- 가장 형식적
- 거의 사용 X — 너무 복잡

---

## 9. Siren

```json
{
  "class": ["order"],
  "properties": {"id": "abc", "status": "pending"},
  "entities": [
    {"class": ["item"], "rel": ["item"], "href": "/orders/abc/items/1"}
  ],
  "actions": [
    {
      "name": "pay",
      "method": "POST",
      "href": "/orders/abc/pay"
    }
  ],
  "links": [
    {"rel": ["self"], "href": "/orders/abc"}
  ]
}
```

---

## 10. 단순 HATEOAS — 흔한 패턴

복잡한 표준 없이 단순 link 추가:

```json
{
  "id": 123,
  "name": "Alice",
  "links": {
    "self": "/users/123",
    "orders": "/users/123/orders",
    "edit": "/users/123",
    "delete": "/users/123"
  }
}
```

또는:
```json
{
  "id": 123,
  "name": "Alice",
  "available_actions": [
    {"action": "edit", "url": "/users/123", "method": "PATCH"},
    {"action": "delete", "url": "/users/123", "method": "DELETE"}
  ]
}
```

---

## 11. Link 헤더 (대안)

응답 본문 대신 HTTP Link 헤더:

```http
HTTP/1.1 200 OK
Link: </users/123/orders>; rel="orders",
      </users/123>; rel="self"
```

### Pagination
```http
Link: </users?page=2>; rel="next",
      </users?page=10>; rel="last",
      </users?page=1>; rel="first",
      </users?page=4>; rel="prev"
```

→ GitHub / RFC 5988 — 페이지네이션의 표준.

---

## 12. 실제 RESTful API 의 HATEOAS

### GitHub API
```json
{
  "id": 1,
  "url": "https://api.github.com/users/octocat",
  "repos_url": "https://api.github.com/users/octocat/repos",
  ...
}
```

→ 일부 link 만 — 진정한 HATEOAS X.

### Stripe API
- Link 거의 X
- ID 만 — 클라가 URL 구성

### PayPal API
- HATEOAS 활성
- 결제 흐름 — 각 단계 link 응답

---

## 13. 함정

### 함정 1 — 모든 곳에 HATEOAS
복잡 — 응답 크기 / 개발 비용.

### 함정 2 — 표준 없이
HAL / JSON:API 등 표준 채택 권장. 자체는 일관성 어려움.

### 함정 3 — 클라가 link 의 의미 모름
"이 link 가 무엇" — 의미 정의 (Profile / Schema / 문서) 필요.

### 함정 4 — OpenAPI 와 충돌
OpenAPI 는 정적 contract — HATEOAS 의 동적성과 trade-off.

### 함정 5 — GraphQL / gRPC 와 비교
명시적 schema 가 더 명확. HATEOAS 의 가치 ↓.

---

## 14. 학습 자료

- "HATEOAS — Roy Fielding" 원문
- HAL — http://stateless.co/hal_specification.html
- JSON:API — https://jsonapi.org/
- RFC 5988 (Link 헤더)
- "Why HATEOAS is dead" 논쟁

---

## 15. 관련

- [[rest]] — REST hub
- [[rest-principles]] — Uniform Interface 의 4 sub-원칙
- [[resource-design]] — URI
- [[../headers/response-headers]] — Link 헤더
