---
title: "DELETE — 삭제"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:52:00+09:00
tags:
  - network
  - http
  - delete
---

# DELETE — 삭제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 멱등 / Soft delete / Body 의 의미 |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ❌ |
| **멱등** | ✅ |
| **캐시 가능** | ❌ |
| **Body** | △ (의미 정의 X — 권장 X) |
| **도입** | HTTP/1.1 |

---

## 1. 한 줄 정의

URI 가 가리키는 **자원을 삭제**. 멱등 — 두 번 호출해도 결과 동일 (이미 삭제됨).

---

## 2. 요청 / 응답

```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer ...

```

```http
HTTP/1.1 204 No Content
```

또는:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"deleted":true,"id":123}
```

---

## 3. 멱등성 — 왜 ✅

```
DELETE /users/123 → 200/204 (삭제됨)
DELETE /users/123 → 404 또는 204 (이미 삭제됨)
DELETE /users/123 → 404 또는 204
```

**상태**는 동일 (= 자원 없음). 응답 코드는 다를 수 있지만 효과는 멱등.

### 함정 — 응용의 부수효과
```
DELETE /users/123 → 사용자 삭제 + 감사 로그 기록
DELETE /users/123 → 로그 한 번 더?  ← 비멱등 부수효과
```

→ 응용이 멱등 보장 책임.

---

## 4. Soft Delete vs Hard Delete

### 4.1 Hard Delete
```sql
DELETE FROM users WHERE id = 123;
```
완전 삭제. 복구 X.

### 4.2 Soft Delete (DB 패턴)
```sql
UPDATE users SET deleted_at = NOW() WHERE id = 123;
```
- 데이터 보존, 일반 query 제외
- 복구 / 감사 / 외래 키 유지
- 대부분 모던 응용이 사용

### 4.3 HTTP 의 표현은 동일
```
DELETE /users/123 → 204
GET /users/123 → 404
```

→ 사용자에게는 "삭제됨" 으로 보임. 내부 DB 가 hard / soft 는 무관.

---

## 5. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 삭제 + body |
| **202 Accepted** | 비동기 삭제 |
| **204 No Content** | 삭제 (body X) |
| **400 Bad Request** | 잘못된 요청 |
| **401 / 403** | 인증 / 인가 |
| **404 Not Found** | 자원 없음 (idempotent 면 204 도 OK) |
| **409 Conflict** | 의존성 (외래 키) |
| **410 Gone** | 영구 삭제됨 (404 의 더 명확한 버전) |

### 404 vs 204 의 선택

```
DELETE /users/123 (없는 자원)
→ 404 Not Found    ← "없다" 명시 (RESTful)
→ 204 No Content   ← idempotent 강조 (재시도 안전)
```

응용 정책. AWS S3 는 204, GitHub 는 404.

---

## 6. DELETE 의 Body

```http
DELETE /users/123 HTTP/1.1
Content-Type: application/json

{"reason": "user requested"}
```

- RFC 9110: "의미 정의 X — 권장 X"
- 일부 응용 (감사 로그) 에서 사용
- 미들박스가 body 폐기 가능
- Elasticsearch — DELETE body 일부 사용

→ **권장 X**. 메타 정보는 헤더 / Query string.

---

## 7. Conditional DELETE

```http
DELETE /users/123 HTTP/1.1
If-Match: "v1"

→ 200 OK 또는 412 Precondition Failed
```

낙관적 잠금 — 다른 클라가 갱신했으면 거부.

---

## 8. Cascading Delete

자원의 의존성:
```
DELETE /users/123
→ 이 사용자의 orders, comments, sessions 모두?
```

### 옵션
- **자동 cascade** (DB ON DELETE CASCADE) — 위험
- **명시적 cascade** (`?cascade=true`)
- **선결조건** — 의존성 있으면 409 Conflict
- **Soft delete** — 의존성 유지, 사용자만 비활성

응용 결정.

---

## 9. Bulk Delete

```
DELETE /users?ids=1,2,3        ← 비표준
DELETE /users (body: [1,2,3])  ← body 위험
POST /users/bulk-delete        ← 안전 (REST 위반이지만 현실)
```

표준 X — 응용 별 정책.

---

## 10. curl 예

```bash
# 기본
curl -X DELETE https://api.example.com/users/123 \
  -H "Authorization: Bearer TOKEN"

# 조건부
curl -X DELETE https://api.example.com/users/123 \
  -H "If-Match: \"v1\""

# body 포함 (권장 X)
curl -X DELETE https://api.example.com/users/123 \
  -H "Content-Type: application/json" \
  -d '{"reason":"user requested"}'
```

---

## 11. WebDAV — Collection Delete

```http
DELETE /folder/ HTTP/1.1
```

폴더 + 내부 모두 삭제. Depth 헤더로 제어 가능 (WebDAV 확장).

---

## 12. 함정

### 함정 1 — GET 으로 삭제
```
GET /users/123/delete   ← 절대 X
```

검색 봇 / 프리페치 → 의도 X 삭제. **DELETE 사용**.

### 함정 2 — 인증 / 인가 없이
DELETE 는 위험 — 항상 strong auth.

### 함정 3 — 외래 키 / 의존성 무시
DB cascade 가 의도 X 삭제 일으킬 수 있음.

### 함정 4 — 멱등의 부수효과
감사 로그 / 알림 — 중복 발생 위험. Idempotency-Key 또는 dedup.

### 함정 5 — Body 의존
미들박스가 폐기 — body 정보 사용 X.

### 함정 6 — Soft delete 의 검색
"삭제된 자원" 검색 / 필터 시 명시. 잘못된 query 가 deleted 포함.

### 함정 7 — CSRF
DELETE 도 CSRF 대상. SameSite / Token 필수.

---

## 13. 학습 자료

- **RFC 9110** Section 9.3.5
- MDN DELETE
- "Soft Delete Pattern" — Microsoft Patterns

---

## 14. 관련

- [[methods]] — Methods hub
- [[put]], [[patch]] — 갱신과 비교
- [[idempotency-safety]]
- [[../security/security]] — CSRF
