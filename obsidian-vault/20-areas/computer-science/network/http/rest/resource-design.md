---
title: "Resource Design — URI / Naming"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:45:00+09:00
tags:
  - network
  - http
  - rest
  - uri
  - naming
---

# Resource Design — URI / Naming

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | URI 설계 / Collection vs Singleton / Nested / Filter |

**[[rest|↑ REST]]** · **[[../http|↑↑ HTTP]]**

---

## 1. URI 의 원칙

### 1.1 명사 — 동사 X

```
좋음:
GET    /users
POST   /users
GET    /users/123
DELETE /users/123

나쁨:
POST   /getUser?id=123       ← 동사
POST   /deleteUser/123       ← 동사
GET    /createOrder?...      ← 동사 + 잘못된 메서드
```

### 1.2 복수형 — 일관

```
좋음 (복수):
/users
/users/123
/orders
/orders/abc

피하기 (혼합):
/user    /orders     ← 혼란
```

### 1.3 Plain 영문 — 소문자 + 하이픈

```
좋음:
/users/123/order-history
/api/v1/users

나쁨:
/Users/123/OrderHistory   ← 대소문자
/users/123/order_history  ← 밑줄 (URL 권장 X)
/users/123/orderHistory   ← camelCase
```

### 1.4 길이
- 너무 깊은 nesting X (최대 2-3 수준)
- 8 KB URL 한계 (브라우저 별)

---

## 2. Collection vs Singleton

### Collection — 복수형
```
/users           ← 모든 사용자
/users/123       ← 1 명
/users           POST → 생성
/users/123       GET / PUT / DELETE
```

### Singleton — 단수 (개념적 유일)
```
/me              ← 현재 사용자
/profile         ← 사용자의 프로필
/settings        ← 시스템 설정
/cart            ← 사용자의 장바구니
```

### 구분
- 여러 인스턴스 가능 — Collection
- 유일 (사용자 별 1, 시스템 1) — Singleton

---

## 3. Nested Resources

### 좋은 nesting (1-2 수준)

```
GET /users/123/orders
POST /users/123/orders
GET /orders/abc/items
GET /orders/abc/items/1
```

### 너무 깊은 nesting

```
GET /companies/1/departments/2/teams/3/members/4/projects/5  ← 너무 깊음
```

### 해결 — Flatten + Query

```
GET /projects?member=4
GET /members/4
```

---

## 4. Filter / Sort / Pagination

### Filter — Query string

```
GET /users?role=admin
GET /users?role=admin&active=true
GET /products?category=electronics&price_min=100&price_max=500
GET /orders?status=pending,processing      ← 콤마로 multi
```

### Sort

```
GET /users?sort=created_at                 ← 오름차순
GET /users?sort=-created_at                ← 내림차순 (- prefix)
GET /users?sort=name,-created_at           ← 다중
```

### Pagination

#### Offset
```
GET /users?page=1&limit=20
GET /users?offset=0&limit=20
```

#### Cursor (큰 데이터)
```
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20
```

→ DB 인덱스 사용 — `LIMIT 20 OFFSET 1000000` 느림.

자세히 → [[../performance/performance]]

### Sparse Fieldsets — 필요 필드만

```
GET /users/123?fields=name,email
```

→ 응답 크기 ↓.

### Embedded — 관련 자원 포함

```
GET /users/123?include=orders,addresses
```

```json
{
  "id": 123,
  "name": "Alice",
  "orders": [...],         ← embedded
  "addresses": [...]
}
```

→ N+1 query 방지 — 한 번에.

---

## 5. RPC-style Actions

REST 의 자원 모델로 표현 어려운 경우 — 실용적 절충.

### 옵션 1 — Sub-resource

```
POST /users/123/activate          ← /users/{id}/{action}
POST /orders/abc/cancel
POST /posts/xyz/publish
```

### 옵션 2 — State 자원

```
PUT /users/123/state             ← {state: "active"}
PUT /orders/abc/status            ← {status: "cancelled"}
```

### 옵션 3 — 명시적 RPC

```
POST /api/rpc/sendEmail
POST /api/jobs/start
```

→ RESTful 보다 명확하면 OK. 일관 유지.

---

## 6. 표준 패턴

### 6.1 인증된 사용자

```
GET /me
GET /users/me
GET /users/self
```

→ "current user" 의미. 명시 ID 불필요.

### 6.2 검색

```
GET /users?q=alice
GET /search?q=alice&type=user
GET /users/search?q=alice
```

→ 검색은 별도 자원 또는 query string. 큰 query 면 POST.

### 6.3 Bulk

```
POST /users/bulk-create     ← body: 여러 사용자
POST /users/bulk-delete     ← body: 여러 ID

PATCH /users (body: 변경 사항 array)
```

→ 비표준 — 응용 정책.

### 6.4 File Upload

```
POST /uploads                multipart/form-data
PUT /uploads/abc            (S3 style)
POST /users/123/avatar      multipart
```

### 6.5 Long-running

```
POST /jobs                   → 202 Accepted + Location: /jobs/abc
GET /jobs/abc               → status 폴링
DELETE /jobs/abc             → 취소
```

자세히 → [[../status-codes/2xx-success]]

---

## 7. Versioning

자세히 → [[api-versioning]]

### URL
```
/api/v1/users
/api/v2/users
```

### Header
```
GET /users
Accept: application/vnd.api.v2+json
```

### Query
```
GET /users?version=2
```

---

## 8. URI 좋은 예 / 나쁜 예

### 좋은 ✅

```
GET    /api/v1/users
GET    /api/v1/users/123
POST   /api/v1/users
PATCH  /api/v1/users/123
DELETE /api/v1/users/123

GET    /api/v1/users/123/orders
GET    /api/v1/orders?status=pending&sort=-created_at&page=1&limit=20

POST   /api/v1/orders/abc/cancel       ← Sub-action OK
GET    /api/v1/me
```

### 나쁜 ❌

```
POST   /api/createUser                ← 동사
GET    /api/v1/getUser?id=123        ← 동사 + 잘못된 컬렉션
GET    /api/users/123/cars/red       ← color 가 path
GET    /v1/User/123                  ← 대소문자 + 단수
GET    /api/users.getById?id=123     ← RPC 스타일
POST   /api/v1/users/123/delete      ← 동사 + DELETE 메서드 사용 X
```

---

## 9. 함정

### 함정 1 — 깊은 nesting
```
/companies/1/depts/2/teams/3/members/4/projects/5  ← 너무 깊음
```

해결 — Flatten + 외래 키 ID.

### 함정 2 — 인증된 user 의 ID 명시
```
GET /users/123/profile (자기 자신)
```

→ `/me/profile` — 권한 검증 단순.

### 함정 3 — 동사 사용
`createUser`, `getUser` — 명사 + 메서드.

### 함정 4 — 일관성 부족
- 어떤 곳은 복수, 어떤 곳은 단수
- 어떤 곳은 camelCase, 어떤 곳은 hyphen
- 일관 유지

### 함정 5 — Trailing slash
`/users` vs `/users/` — 일관 (보통 trailing slash X).

### 함정 6 — Pagination 누락
큰 데이터 — 응답 폭증. 항상 `limit` + 기본값.

### 함정 7 — Filter 의 SQL Injection
Query param → 직접 SQL X. ORM / Prepared statement.

---

## 10. 학습 자료

- "RESTful API Design" — Vinay Sahni
- **REST API Design Rulebook** (Mark Massé)
- Stripe / GitHub / Twilio API 가이드 (모범 사례)
- Google API Design Guide

---

## 11. 관련

- [[rest]] — REST hub
- [[rest-principles]] — Uniform Interface
- [[api-versioning]] — 버전 관리
- [[../methods/methods]] — 메서드 ↔ CRUD
- [[../status-codes/status-codes]]
