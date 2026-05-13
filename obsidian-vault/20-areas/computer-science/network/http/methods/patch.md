---
title: "PATCH — 부분 갱신"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:49:00+09:00
tags:
  - network
  - http
  - patch
---

# PATCH — 부분 갱신

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RFC 5789 / JSON Merge / JSON Patch / 멱등 X |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ❌ |
| **멱등** | ❌ (보장 X, 응용 결정) |
| **캐시 가능** | ❌ |
| **Body** | ✅ |
| **도입** | RFC 5789 (2010) |

---

## 1. 한 줄 정의

자원의 **일부 필드만 갱신**. PUT 의 "전체" 와 대비. RFC 5789 (2010).

---

## 2. 요청 / 응답

```http
PATCH /api/users/123 HTTP/1.1
Content-Type: application/merge-patch+json
Content-Length: 30

{"email":"alice@new.com"}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":123,"name":"Alice","email":"alice@new.com","age":30}
```

→ email 만 변경, 다른 필드 유지.

---

## 3. PATCH 의 Content-Type 표준

### 3.1 JSON Merge Patch (RFC 7396)

`application/merge-patch+json`

```http
원본: {"id":1,"name":"Alice","age":30,"hobbies":["reading"]}

PATCH:
{"age":31, "email":"alice@new.com", "hobbies":null}

결과:
{"id":1,"name":"Alice","age":31,"email":"alice@new.com","hobbies":null(=삭제)}
```

#### 규칙
- 필드 명시 → 갱신
- null → 삭제
- 명시 안 함 → 유지

#### 한계
- 배열 부분 변경 X (전체 교체)
- null 의 의미 모호

### 3.2 JSON Patch (RFC 6902)

`application/json-patch+json`

```http
PATCH /users/123
Content-Type: application/json-patch+json

[
  {"op": "replace", "path": "/age", "value": 31},
  {"op": "add",     "path": "/email", "value": "alice@new.com"},
  {"op": "remove",  "path": "/oldField"},
  {"op": "copy",    "from": "/firstName", "path": "/displayName"},
  {"op": "move",    "from": "/temp",  "path": "/permanent"},
  {"op": "test",    "path": "/version", "value": 2}
]
```

#### 6 가지 op
- **add** — 추가
- **remove** — 삭제
- **replace** — 교체
- **copy** — 복사
- **move** — 이동
- **test** — 값 검증 (실패 시 patch 전체 중단)

#### JSON Pointer (RFC 6901)
- `/users/0/name` — 배열 인덱스
- `/items/-` — 배열 끝 (append)

### 3.3 XML Patch (RFC 5261)

XML 환경 — 거의 사용 X.

### 3.4 application/json (커스텀)

대부분 API 가 표준 X — 자체 정의:
```http
PATCH /users/123
Content-Type: application/json

{"email":"alice@new.com"}
```

→ 보통 JSON Merge Patch 와 같지만 명시 표준 X.

---

## 4. PATCH vs PUT

| 측면 | PUT | PATCH |
| --- | --- | --- |
| 범위 | 전체 | 부분 |
| 본문 | 자원 전체 | 변경만 |
| 멱등 | ✅ | ❌ (보장 X) |
| 누락 필드 | 삭제 / 기본 | 유지 |
| 본문 크기 | 큼 | 작음 |
| 대역폭 | 큼 | 작음 |

---

## 5. 왜 멱등 X (보장 X)

```
PATCH [{"op":"add", "path":"/counter/-", "value":1}]
첫 호출: counter = [1]
두 번째: counter = [1, 1]
세 번째: counter = [1, 1, 1]
```

→ 응용이 멱등성을 명시적으로 보장해야.

### JSON Merge Patch 는 보통 멱등
```
PATCH {"email":"new"} 두 번 → 같은 결과
```

### JSON Patch 의 op 에 따라
- `replace` — 멱등
- `add` (배열) — **멱등 X**
- `remove` — 멱등

→ "PATCH 멱등 X" 는 **보장 X**. 응용이 멱등 설계 가능.

---

## 6. Conditional PATCH

PUT 과 동일:
```http
PATCH /users/123
If-Match: "v1"
Content-Type: application/merge-patch+json

{"email":"..."}
```

→ 동시성 제어.

---

## 7. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 갱신 + body |
| **204 No Content** | 갱신, body X |
| **400 Bad Request** | 검증 실패 |
| **409 Conflict** | 충돌 |
| **412 Precondition Failed** | If-Match 실패 |
| **415 Unsupported Media Type** | Content-Type 미지원 |
| **422 Unprocessable Entity** | 의미 오류 (JSON Patch test 실패 등) |

---

## 8. curl 예

```bash
# JSON Merge Patch
curl -X PATCH https://api.com/users/123 \
  -H "Content-Type: application/merge-patch+json" \
  -d '{"email":"alice@new.com"}'

# JSON Patch (RFC 6902)
curl -X PATCH https://api.com/users/123 \
  -H "Content-Type: application/json-patch+json" \
  -d '[{"op":"replace","path":"/email","value":"alice@new.com"}]'
```

---

## 9. 사용 사례

### 9.1 SaaS API
- Notion API — `PATCH /pages/{id}` (블록 갱신)
- Stripe — `POST /customers/{id}` (Stripe 는 historic 이유로 POST)
- GitHub — `PATCH /repos/{owner}/{repo}` (저장소 설정)

### 9.2 Kubernetes
```bash
kubectl patch deployment my-app -p '{"spec":{"replicas":3}}'
# → JSON Merge Patch 기본

kubectl patch deployment my-app --type json -p '[{"op":"replace","path":"/spec/replicas","value":3}]'
# → JSON Patch
```

### 9.3 GraphQL Mutation
- 부분 갱신과 비슷한 사상
- HTTP PATCH 와 별개

---

## 10. 함정

### 함정 1 — PUT 으로 PATCH 시도
누락 필드 삭제됨.

### 함정 2 — null 의 의미
JSON Merge Patch 의 null = 삭제. JSON 의 의도된 null 값과 혼동.

→ "삭제 명시" 필요 시 JSON Patch.

### 함정 3 — Content-Type 미지정
서버가 처리 모름 → 415.

### 함정 4 — 멱등 가정
PATCH 가 멱등이라 가정 → 재시도 시 중복.

→ Idempotency-Key 사용.

### 함정 5 — Batch / 트랜잭션
JSON Patch 가 array 라 의미상 atomic 보이지만 응용 구현 의존.

### 함정 6 — 부분 검증
일부 필드만 검증 → 다른 필드와 불일치 가능.

### 함정 7 — 보안 / 권한
필드별 권한 검증 필요 (admin 만 role 변경 등).

---

## 11. 학습 자료

- **RFC 5789** (PATCH), **RFC 6902** (JSON Patch), **RFC 7396** (JSON Merge Patch)
- **RFC 6901** (JSON Pointer)
- MDN PATCH
- "JSON Patch vs Merge Patch" 가이드

---

## 12. 관련

- [[methods]] — Methods hub
- [[put]] — 전체 갱신
- [[idempotency-safety]]
- [[../headers/entity-headers]] — Content-Type 표준
