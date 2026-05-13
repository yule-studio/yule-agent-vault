---
title: "REST 의 6 가지 제약 (Fielding 2000)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:40:00+09:00
tags:
  - network
  - http
  - rest
  - principles
---

# REST 의 6 가지 제약 (Fielding 2000)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 6 제약 깊이 |

**[[rest|↑ REST]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 6 제약 한눈

| # | 제약 | 의미 |
| --- | --- | --- |
| 1 | **Client-Server** | 분리 |
| 2 | **Stateless** | 상태 X |
| 3 | **Cacheable** | 캐시 가능 |
| 4 | **Layered System** | 계층화 |
| 5 | **Uniform Interface** | 일관된 인터페이스 |
| 6 | **Code on Demand** (옵션) | 코드 전달 |

---

## 2. Constraint 1 — Client-Server

### 의미
- 클라이언트 / 서버 분리
- 독립 진화

### 효과
- UI 변경이 서버에 영향 X
- 서버 확장이 UI 에 영향 X
- 다양한 클라 (웹 / 모바일 / IoT) 가 같은 API

### 위반 예
- 클라 / 서버가 같은 코드 베이스 (server-rendered 의 일부)
- 결합 강함

---

## 3. Constraint 2 — Stateless

### 의미
- **모든 요청이 자기 정보 포함**
- 서버가 클라 상태 X 저장

### 좋은 예
```http
GET /api/me HTTP/1.1
Authorization: Bearer eyJhbGc...

→ JWT 가 user 정보 — 서버는 토큰 검증만
```

### 나쁜 예
```
Step 1: POST /login → 서버에 SessionID 저장
Step 2: GET /step1 → 서버가 "이 사용자 step 1 끝났음" 기억
Step 3: GET /step2 → 서버의 상태에 의존
```

→ 서버 인스턴스 별 상태 → 스케일링 어려움.

### 효과
- Stateless = 스케일링 쉬움 (모든 인스턴스가 동등)
- LB 의 round-robin OK
- 캐시 가능

### 미세
- "Session cookie + Session Store" 는 RESTful?
  - 응용 입장 stateless (cookie = ID 만)
  - 엄밀히는 Server-side state — 논쟁
  - 실용 — 둘 다 RESTful 으로 봄

---

## 4. Constraint 3 — Cacheable

### 의미
- 응답에 캐시 가능 여부 명시
- HTTP 의 `Cache-Control`, `ETag` 활용

### RESTful 한 캐싱
```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
ETag: "v1"
```

자세히 → [[../caching/caching]]

### 효과
- Latency ↓
- Server load ↓
- 네트워크 효율 ↑

### 위반
- 모든 응답에 `Cache-Control: no-store` (의도 X 시)
- ETag 무시

---

## 5. Constraint 4 — Layered System

### 의미
- 클라가 직접 서버 vs proxy 구분 X
- LB / CDN / API Gateway 투명

### 효과
```
Client → CDN → API Gateway → LB → App Server → DB Cache → DB

각 계층 독립
새 계층 추가가 클라에 영향 X
```

### 위반
- 클라가 "직접 origin server" 가정
- IP / Host 하드코딩

---

## 6. Constraint 5 — Uniform Interface ⭐

REST 의 핵심 제약. 4 sub-원칙:

### 6.1 Identification of Resources

자원이 URI 로 식별:
```
/users/123          ← 사용자
/users/123/orders   ← 주문들
/posts/abc          ← 게시물
```

### 6.2 Manipulation through Representations

자원 자체가 아닌 **표현** 을 주고받음:
```http
GET /users/123
Accept: application/json
↓ JSON 표현

GET /users/123
Accept: application/xml
↓ XML 표현
```

→ 같은 자원의 여러 표현.

### 6.3 Self-descriptive Messages

각 메시지가 처리 정보 포함:
```http
POST /users HTTP/1.1
Content-Type: application/json    ← 본문 의미
Accept: application/json          ← 응답 원함
Authorization: Bearer ...         ← 인증
```

응답:
```http
HTTP/1.1 201 Created             ← 상태
Location: /users/123              ← 위치
Content-Type: application/json    ← 본문 의미
{...}
```

→ 누구나 메시지 보고 의미 파악 가능.

### 6.4 HATEOAS — Hypermedia as State Engine

응답에 다음 가능한 action 의 link:
```json
{
  "id": 123,
  "status": "pending",
  "_links": {
    "self": {"href": "/orders/123"},
    "cancel": {"href": "/orders/123/cancel", "method": "POST"},
    "pay": {"href": "/orders/123/pay", "method": "POST"}
  }
}
```

→ 클라가 다음 action 학습 (URL 하드코딩 X). 자세히 → [[hateoas]]

---

## 7. Constraint 6 — Code on Demand (옵션)

### 의미
- 서버가 클라에 코드 전달 (JS / WebAssembly)
- 클라가 실행
- 옵션 — REST 강제 X

### 예
- HTML + `<script src="...">` → JS 실행
- WASM 모듈 fetch + 실행

### 효과
- 클라 기능 확장
- 단일 표준 클라 (브라우저) 의 강력함

---

## 8. RESTful API 체크리스트

```
1. 자원 명사 URI?            ✓ /users/123 (not /getUser)
2. HTTP 메서드 의미?         ✓ GET (read), POST (create), ...
3. 상태 코드 의미?           ✓ 200 / 201 / 404 / ...
4. Content-Type 명시?       ✓
5. Cache-Control / ETag?    ✓
6. Authorization 표준?       ✓ Bearer / Basic
7. Stateless?               ✓ (Session 만 OK)
8. Layered (proxy 투명)?    ✓
9. Self-descriptive?        ✓
10. HATEOAS?                △ (선택)
```

---

## 9. 자주 위반되는 제약

### 9.1 Stateless 위반
- 서버가 클라별 multi-step 상태
- 해결 — 클라가 보내거나 JWT

### 9.2 Uniform Interface 위반
- RPC 스타일 (`POST /createUser`)
- 메서드 의미 무시 (`POST` 로 모든 것)
- 해결 — RESTful 규칙

### 9.3 Cacheable 위반
- 모든 응답 `no-store`
- 해결 — 가능한 곳 캐시

---

## 10. Fielding 의 비판

> "Most REST APIs aren't REST" — Fielding 자신

대부분 응용:
- Maturity Level 2 (메서드 / 상태 코드 만)
- HATEOAS 거의 X
- 그래도 "REST" 라 부름

→ Fielding 의 엄격한 정의 vs 실용 차이.

---

## 11. 모범 사례

### 11.1 일관성
- 모든 endpoint 같은 패턴
- 같은 에러 형식
- 같은 페이지네이션

### 11.2 명확한 URI
```
/users/123/orders/456
```
보다
```
/getUserOrder?u=123&o=456
```

### 11.3 표준 응답 + 에러
- RFC 7807 Problem Details
- JSON:API 등

### 11.4 적절한 상태 코드
- 201 (생성) / 204 (성공 + body X) / 409 (충돌) / 422 (검증)

### 11.5 캐시
- 정적 자원 + 변경 적은 데이터 적극 캐시

---

## 12. 학습 자료

- **Fielding's Dissertation** — Ch. 5
- **REST in Practice** — Webber
- "Roy Fielding's REST" — 원문 강조

---

## 13. 관련

- [[rest]] — REST hub
- [[resource-design]] — URI 설계
- [[hateoas]] — Level 3
- [[../caching/caching]] — Cacheable
- [[../methods/methods]] — Uniform Interface
