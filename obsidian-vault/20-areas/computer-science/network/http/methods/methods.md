---
title: "HTTP 메서드 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:35:00+09:00
tags:
  - network
  - http
  - methods
---

# HTTP 메서드 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 9 개 메서드 + 멱등 / 안전 / 캐시 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 메서드 한눈

| 메서드 | 의미 | 안전 | 멱등 | 캐시 | Body | 도입 |
| --- | --- | --- | --- | --- | --- | --- |
| **GET** | 조회 | ✅ | ✅ | ✅ | ❌ | 0.9 |
| **POST** | 생성 / 처리 | ❌ | ❌ | △ | ✅ | 1.0 |
| **PUT** | 전체 갱신 | ❌ | ✅ | ❌ | ✅ | 1.1 |
| **PATCH** | 부분 갱신 | ❌ | ❌ | ❌ | ✅ | RFC 5789 |
| **DELETE** | 삭제 | ❌ | ✅ | ❌ | △ | 1.1 |
| **HEAD** | 헤더만 | ✅ | ✅ | ✅ | ❌ | 1.0 |
| **OPTIONS** | 허용 / CORS | ✅ | ✅ | ❌ | ❌ | 1.1 |
| **TRACE** | 진단 | ✅ | ✅ | ❌ | ❌ | 1.1 |
| **CONNECT** | Proxy tunnel | ❌ | ❌ | ❌ | ❌ | 1.1 |

자세히:
- [[get]], [[post]], [[put]], [[patch]], [[delete]]
- [[head]], [[options]], [[trace-connect]]
- [[idempotency-safety]] — 속성 깊이

---

## 2. 메서드의 3 가지 속성

### 2.1 Safe (안전)
- 서버 상태 변경 X
- GET, HEAD, OPTIONS, TRACE
- 로봇이 마음대로 호출 OK

### 2.2 Idempotent (멱등)
- N 번 호출 = 1 번 호출 결과
- GET, HEAD, OPTIONS, PUT, DELETE
- 네트워크 실패 시 재시도 안전

### 2.3 Cacheable (캐시 가능)
- 응답 캐시 OK
- GET, HEAD (조건부 POST)

→ 자세히 [[idempotency-safety]]

---

## 3. 의미 매트릭스

```
        | 안전 | 멱등 | 캐시 |
GET     |  O   |  O   |  O   |   ← 가장 안전
HEAD    |  O   |  O   |  O   |
OPTIONS |  O   |  O   |  X   |
TRACE   |  O   |  O   |  X   |   ← 안전하지만 보안 위험으로 차단
PUT     |  X   |  O   |  X   |   ← 멱등 (전체 교체)
DELETE  |  X   |  O   |  X   |   ← 멱등 (이미 삭제됨이라도 OK)
POST    |  X   |  X   |  △   |
PATCH   |  X   |  X   |  X   |
CONNECT |  X   |  X   |  X   |
```

---

## 4. CRUD 매핑

| CRUD | HTTP |
| --- | --- |
| Create | POST `/users` |
| Read | GET `/users/123` |
| Update (전체) | PUT `/users/123` |
| Update (부분) | PATCH `/users/123` |
| Delete | DELETE `/users/123` |

자세히 → [[../rest/rest]]

---

## 5. WebDAV 추가 메서드 (참고)

| 메서드 | 용도 |
| --- | --- |
| **PROPFIND** | 속성 조회 |
| **PROPPATCH** | 속성 수정 |
| **MKCOL** | 컬렉션 생성 |
| **COPY** | 복사 |
| **MOVE** | 이동 |
| **LOCK** / **UNLOCK** | 잠금 |

원격 파일 시스템 (NextCloud, OwnCloud).

---

## 6. 면접 / 토픽

1. **GET vs POST** — 안전 / 멱등 / 본문 / 캐시.
2. **PUT vs PATCH** — 전체 교체 vs 부분.
3. **DELETE 가 멱등인 이유**.
4. **TRACE 가 차단되는 이유** — XST (Cross-Site Tracing).
5. **OPTIONS 의 CORS 역할**.
6. **POST 가 캐시 가능한 경우** (`Cache-Control: public`).

---

## 7. 학습 자료

- **RFC 9110** Section 9 (Methods)
- MDN HTTP Methods
- **HTTP: The Definitive Guide** Ch. 3

---

## 8. 관련

- [[../http]] — HTTP hub
- [[../status-codes/status-codes]] — 함께 사용
- [[../rest/rest]] — CRUD 매핑
