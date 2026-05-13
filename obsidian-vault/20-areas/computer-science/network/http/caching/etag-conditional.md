---
title: "ETag & Conditional Requests"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:18:00+09:00
tags:
  - network
  - http
  - caching
  - etag
  - conditional
---

# ETag & Conditional Requests

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ETag (strong/weak) / If-None-Match / If-Match / 304 |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

**자원의 fingerprint** — 자원이 변경되면 ETag 변경. 조건부 GET (캐시) + 조건부 PUT
(낙관적 잠금) 의 핵심.

---

## 2. ETag 형식

```http
ETag: "abc123"                                  (strong)
ETag: "v1.0-abc123"
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"  (해시)
ETag: W/"weak-abc"                              (weak)
```

### 따옴표
- 항상 `"..."` 안
- weak 면 `W/` prefix

---

## 3. Strong vs Weak ETag

### Strong
- byte 단위 동일
- gzip 압축 vs 평문 — 다른 ETag
- Range request 시 strong 만

### Weak
- 의미 동일 — byte 다를 수 있음
- gzip 압축 / 광고 변동 등 OK
- `W/` prefix

```http
ETag: "abc"        ← strong
ETag: W/"abc"      ← weak
```

→ If-Match (PUT) 은 **strong 만**. If-None-Match (GET) 은 weak 도 OK.

---

## 4. 조건부 GET — If-None-Match

```http
[첫 요청]
GET /api/users/123 HTTP/1.1

HTTP/1.1 200 OK
ETag: "v1"
Cache-Control: max-age=300
{...body...}


[캐시 만료 후 검증]
GET /api/users/123 HTTP/1.1
If-None-Match: "v1"

HTTP/1.1 304 Not Modified
ETag: "v1"
Cache-Control: max-age=300
(body 없음)


[변경 시]
GET /api/users/123 HTTP/1.1
If-None-Match: "v1"

HTTP/1.1 200 OK
ETag: "v2"
{...new body...}
```

### 의미
- 클라가 보유한 ETag = 서버의 현재 ETag → 304 (body 절약)
- 다름 → 200 + 새 body

### 여러 ETag

```http
If-None-Match: "v1", "v2"
```

→ 둘 중 하나 매칭이면 304.

### Wildcard

```http
If-None-Match: *
```

→ "어떤 ETag 든 매칭" — "자원 존재하면 304".

---

## 5. 조건부 PUT — If-Match (낙관적 잠금)

```http
GET /api/users/123
→ 200 OK, ETag: "v1"

(사용자 편집)

PUT /api/users/123
If-Match: "v1"
{...edited...}

→ 200 OK, ETag: "v2"     (성공)
또는
→ 412 Precondition Failed (옛 버전 — 다른 사용자가 갱신)
```

### 의미
- "마지막 본 버전" 일 때만 갱신
- 동시 편집 충돌 방지

### Lost Update 방지

```
사용자 A: GET → ETag: "v1"
사용자 B: GET → ETag: "v1"
사용자 A: PUT If-Match "v1" → 200 → ETag: "v2"
사용자 B: PUT If-Match "v1" → 412 (A 가 갱신)
사용자 B: GET → 최신 받아 다시 시도
```

### 강제 — 428 Precondition Required

```http
PUT /api/users/123
(If-Match 없이)

HTTP/1.1 428 Precondition Required
{"error": "If-Match header required for updates"}
```

→ 응용이 조건부 PUT 강제.

---

## 6. If-None-Match: * — 안전한 Create

```http
PUT /api/users/123
If-None-Match: *

→ 201 Created   (자원 없었음)
또는
→ 412 Precondition Failed (이미 있음)
```

→ "없을 때만 생성" — atomic create.

---

## 7. ETag 생성 전략

### 7.1 Content Hash

```python
import hashlib
etag = '"' + hashlib.sha256(content).hexdigest() + '"'
```

- 가장 정확
- 큰 자원은 비용 ↑
- 약한 hash (CRC32) 가능

### 7.2 Version + Timestamp

```python
etag = f'"{user.version}-{user.updated_at.timestamp()}"'
```

- DB 에서 빠름
- weak ETag 으로 표현 가능

### 7.3 Last-Modified 기반

```python
etag = 'W/"' + format_http_date(last_modified) + '"'
```

- Last-Modified 헤더와 중복 — ETag 더 표현력 ↑

### 7.4 inode + size + mtime (정적 파일)

```python
import os
stat = os.stat(path)
etag = f'"{stat.st_ino}-{stat.st_size}-{int(stat.st_mtime)}"'
```

- Apache / Nginx 기본 방식
- 일부 클러스터에서 inode 다름 → 불일치 위험

---

## 8. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 변경됨, 새 body |
| **201 Created** | If-None-Match: * 로 생성됨 |
| **304 Not Modified** | 캐시 valid (body X) |
| **412 Precondition Failed** | If-Match 실패 |
| **428 Precondition Required** | 조건부 요청 요구 |

---

## 9. ETag vs Last-Modified

| 측면 | ETag | Last-Modified |
| --- | --- | --- |
| 표현 | 임의 (해시 등) | 시간 |
| 정밀도 | byte 단위 | 초 단위 |
| 서브-초 변경 | ✅ | ❌ |
| 의미 | content fingerprint | 수정 시간 |
| 우선순위 | 우선 | fallback |

→ **둘 다 보내는 게 표준**. 클라가 우선 ETag 사용, 없으면 Last-Modified.

자세히 → [[last-modified]]

---

## 10. ETag 의 보안 / 프라이버시 이슈

### Tracking
- 일부 사이트가 ETag 를 **고유 ID** 로 사용 (쿠키 차단 우회)
- "ETag tracking" — 브라우저가 캐시 차단 시 사라짐

### 방어
- 브라우저: Incognito / Private 모드
- Safari: ITP (Intelligent Tracking Prevention)
- ETag 가 사용자별 고유 X — 자원 fingerprint 만

---

## 11. 디버깅

```bash
# 첫 요청
curl -i https://example.com/api/users/123
# ETag: "v1"

# 조건부
curl -i -H 'If-None-Match: "v1"' https://example.com/api/users/123
# HTTP/1.1 304 Not Modified

# 강제 변경 (PUT)
curl -i -X PUT -H 'If-Match: "v1"' -d '{...}' https://example.com/api/users/123
```

---

## 12. 함정

### 함정 1 — ETag 의 따옴표 누락
```
ETag: abc         ← 잘못
ETag: "abc"       ← 올바름
```

### 함정 2 — Strong / Weak 혼동
- Strong 만 If-Match (PUT) 에 OK
- Weak 는 byte 단위 동일 가정 X

### 함정 3 — 클러스터의 다른 ETag
인스턴스 별 inode 다름 → 같은 자원이 다른 ETag → 304 안 됨. **명시적 hash** 사용.

### 함정 4 — gzip 의 ETag
- nginx 옛: 압축 후 weak ETag 자동
- 모던: 응용이 ETag 생성, 인코딩 변경 시 갱신

### 함정 5 — ETag 와 Vary
같은 자원이 압축 / 언어 별 다른 ETag — Vary 헤더 필수.

### 함정 6 — 너무 비싼 ETag
큰 파일의 SHA256 — 매 요청 비용. 캐싱 / Last-Modified 만 사용도 OK.

### 함정 7 — ETag 누락 + Last-Modified 만
정밀도 부족 — 같은 초 안에서 변경 시 인식 X.

---

## 13. 학습 자료

- **RFC 9110** Section 8.8 (ETag), 13 (Conditional)
- **RFC 9111** Section 4.3 (Validation)
- MDN ETag
- "Strong vs Weak ETags" — REST API 가이드

---

## 14. 관련

- [[caching]] — Caching hub
- [[cache-control]] — max-age 와 결합
- [[last-modified]] — 시간 기반 검증
- [[vary-header]] — 캐시 키
- [[../headers/response-headers]] — ETag 위치
- [[../status-codes/3xx-redirection]] — 304
- [[../methods/put]] — If-Match
