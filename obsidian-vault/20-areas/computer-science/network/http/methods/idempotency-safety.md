---
title: "Idempotency, Safety, Cacheability"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:05:00+09:00
tags:
  - network
  - http
  - idempotency
  - safety
---

# Idempotency, Safety, Cacheability

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 3 가지 속성 깊이 + Idempotency Key 패턴 |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 3 가지 속성 한눈

| 속성 | 의미 | 결과 |
| --- | --- | --- |
| **Safe** | 서버 상태 변경 X | 로봇 / 프리페치 OK |
| **Idempotent** | N 번 = 1 번 | 재시도 안전 |
| **Cacheable** | 응답 캐시 OK | CDN / 브라우저 |

→ Safe ⊂ Idempotent ⊂ ... (관계 있지만 별개 속성).

---

## 2. 메서드별 매트릭스

| Method | Safe | Idempotent | Cacheable |
| --- | --- | --- | --- |
| GET | ✅ | ✅ | ✅ |
| HEAD | ✅ | ✅ | ✅ |
| OPTIONS | ✅ | ✅ | ❌ |
| TRACE | ✅ | ✅ | ❌ |
| PUT | ❌ | ✅ | ❌ |
| DELETE | ❌ | ✅ | ❌ |
| POST | ❌ | ❌ | △ (명시 시) |
| PATCH | ❌ | ❌ (보장 X) | ❌ |
| CONNECT | ❌ | ❌ | ❌ |

---

# Part A. Safe (안전)

## A.1 정의 (RFC 9110)

> "A request method is considered **safe** if its defined semantics
> are essentially **read-only**".

→ 서버 상태 변경 X. 로그 / 통계 등 부수 효과는 OK (의도 안 한 거).

## A.2 Safe 메서드

- **GET, HEAD, OPTIONS, TRACE**

## A.3 의미

### A.3.1 로봇 / 크롤러
- 안전한 메서드는 마음대로 호출 OK
- `robots.txt` 가 GET 만 제어

### A.3.2 프리페치 (Link Prefetch)
```html
<link rel="prefetch" href="/next-page">
```

→ 브라우저가 GET 자동 호출. 안전 가정.

### A.3.3 캐시 / 프록시
- 응답 캐시 (response cacheable 면)
- 중복 요청 결합 (request coalescing)

## A.4 함정 — GET 으로 상태 변경

```
GET /api/users/123/delete   ← 절대 X
GET /api/like?post=42       ← 사실상 X
```

→ 봇이 모든 페이지 GET → 의도 X 삭제 / 좋아요 폭증.

**해결**:
- 상태 변경 = POST / PUT / DELETE / PATCH
- "Like" 도 `POST /likes`

---

# Part B. Idempotent (멱등)

## B.1 정의

> "Multiple identical requests have the **same effect** as a single request"

→ N 번 호출 = 1 번 호출 (서버 상태 관점).

## B.2 Idempotent 메서드

- **GET, HEAD, OPTIONS, TRACE** — Safe (자동 멱등)
- **PUT** — 전체 교체
- **DELETE** — 이미 삭제됨이라도 OK

## B.3 PUT 의 멱등

```
PUT /users/123 {name: "Alice"}     → state = {name: "Alice"}
PUT /users/123 {name: "Alice"}     → state = {name: "Alice"} (변화 X)
PUT /users/123 {name: "Alice"}     → 동일
```

## B.4 DELETE 의 멱등

```
DELETE /users/123 → state = 없음 (200 / 204)
DELETE /users/123 → 이미 없음 (404 또는 204)
DELETE /users/123 → 동일
```

→ 응답 코드 다르지만 **상태**는 동일.

## B.5 POST / PATCH 가 비멱등 인 이유

### POST
```
POST /orders {amount: 100}
→ 주문 1 생성
POST /orders {amount: 100}
→ 주문 2 생성    ← 두 주문!
```

### PATCH
```
PATCH {op:"add", path:"/counter/-", value:1}
1 번: counter=[1]
2 번: counter=[1,1]
```

(JSON Merge Patch 의 일부는 멱등 — 단순 replace 만.)

## B.6 응용의 부수 효과로 멱등 깨짐

### B.6.1 updated_at 자동 갱신

```
PUT /users/123 → updated_at = NOW()
PUT /users/123 → updated_at = NOW() (다른 시간)
```

→ 외부 관점은 멱등, 내부 DB 는 변화 — RFC 의 멱등은 **외부 의미** 만.

### B.6.2 감사 로그

```
DELETE /users/123 → 삭제 + audit log
DELETE /users/123 → 이미 없음, but audit log 한 번 더?
```

응용이 dedup 책임.

## B.7 Idempotency Key 패턴

POST 를 멱등으로 — 응용 레벨.

```http
POST /api/payments HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{amount: 100, ...}

→ 첫 요청: 처리 후 응답 + (key, response) 저장
→ 같은 key 두 번째: 저장된 응답 그대로 반환 (재처리 X)
```

### 구현
- Key 보통 UUID
- 저장: Redis (24h TTL)
- Stripe, Adyen, AWS API Gateway 가 사용

### 응용 코드 (Python)

```python
def create_payment(req, idempotency_key):
    cached = redis.get(f"idem:{idempotency_key}")
    if cached:
        return cached
    
    response = process_payment(req)
    redis.setex(f"idem:{idempotency_key}", 86400, response)
    return response
```

### 함정
- 같은 key + 다른 body — 정의: 첫 요청 결과 반환 (body 차이 무시 또는 400 거부)
- Key 누수 — 다른 사용자가 같은 key 사용 시 — Auth 와 결합 (`user_id + key`)

## B.8 Conditional Request 와 결합

```http
PUT /users/123
If-Match: "v1"
```

→ 옛 버전일 때만 갱신 (낙관적 잠금). 멱등의 안전한 적용.

---

# Part C. Cacheable (캐시 가능)

## C.1 기본 캐시 가능 메서드

- **GET, HEAD** — 기본 캐시
- **POST** — `Cache-Control` 명시 시만 (드뭎)

## C.2 GET 의 캐시

```http
GET /api/users/123 HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: max-age=300, public
ETag: "v1"

→ 5 분간 캐시
```

## C.3 POST 의 캐시 (가능하지만 드묾)

```http
POST /api/search HTTP/1.1
Cache-Control: max-age=60

{...}

HTTP/1.1 200 OK
Cache-Control: max-age=60
```

→ 비싼 검색 결과 캐시. 실제 사용 적음 (POST URL 이 자원 표현 X).

## C.4 PUT / DELETE / PATCH — 캐시 무효화

```
PUT /users/123 → 200 OK

→ Cache 가 /users/123 의 기존 GET 응답 invalidate
```

자세히 → [[../caching/caching]]

## C.5 응답 코드와 캐시

| 코드 | 기본 캐시 가능? |
| --- | --- |
| 200, 203, 204 | ✅ |
| 206 (Partial) | ✅ (heuristic) |
| 300, 301 | ✅ |
| 302, 307 | ❌ (Cache-Control 명시 시 가능) |
| 404, 405, 410, 414 | ✅ (long-term) |
| 501 | ✅ |
| 나머지 | ❌ (명시 시만) |

---

# Part D. 종합 — 실용 결정 트리

## D.1 메서드 선택

```
1. 자원 변경 X?  → GET (또는 HEAD/OPTIONS)
2. 자원 식별 X (컬렉션에 추가)?  → POST
3. 자원 식별 O, 전체 교체?  → PUT
4. 자원 식별 O, 부분 갱신?  → PATCH
5. 자원 삭제?  → DELETE
```

## D.2 재시도 정책

```
네트워크 timeout 시:
  Safe / Idempotent  → 자동 재시도 OK
  POST              → Idempotency Key + 재시도, 또는 거부
  PATCH             → 응용 결정
```

대부분 HTTP 라이브러리:
- `retry on idempotent only` (default)
- POST 는 명시 시만

## D.3 LB / Proxy 의 retry

```
HAProxy / Nginx:
  GET / HEAD: 안전 재시도
  POST: 기본 X (위험)

명시 (Nginx):
  proxy_next_upstream non_idempotent;  ← 위험
```

---

## 7. 흔한 RFC 위반

### 7.1 GET 으로 상태 변경
구식 사이트의 `?delete=true` — RFC 위반.

### 7.2 POST 로 모든 것
RPC 스타일 — 멱등성 잃음. 재시도 위험.

### 7.3 PUT 으로 부분 갱신
누락 필드 삭제됨. PATCH 사용.

### 7.4 멱등 안 보장하는 멱등
응용 부수 효과 (count 증가) — RFC 의 외부 의미는 멱등이지만 위험.

### 7.5 캐시 무효화 누락
PUT 후 옛 GET 응답 반환 — `Cache-Control: no-cache` 또는 invalidation.

---

## 8. 면접 답변 모범

**Q. GET 이 idempotent 인 이유?**

> GET 은 자원 조회만 의미한다. 같은 GET 을 N 번 호출해도 서버 상태는 변화 없으니
> 결과는 동일하다. 따라서 멱등. 단 응용이 GET 으로 상태 변경 (count 증가 등) 하면
> 사실상 비멱등 — RFC 위반.

**Q. DELETE 가 idempotent 인 이유?**

> 첫 DELETE 는 자원 삭제. 두 번째는 이미 없음 (404) — 하지만 **서버 상태는 동일** (자원 없음).
> 응답 코드는 다를 수 있지만 효과는 멱등.

**Q. POST 를 idempotent 로 만들려면?**

> Idempotency Key 패턴. 클라가 UUID 생성 → 같은 key 의 두 번째 요청은 첫 응답 그대로
> 반환. Stripe, AWS 가 사용.

---

## 9. 학습 자료

- **RFC 9110** Section 9.2 (Safe, Idempotent, Cacheable)
- Stripe Idempotency 가이드
- "REST API 의 멱등성" 시리즈
- MDN HTTP Method 비교

---

## 10. 관련

- [[methods]] — Methods hub
- [[get]], [[post]], [[put]], [[patch]], [[delete]]
- [[../caching/caching]] — Cacheable 깊이
