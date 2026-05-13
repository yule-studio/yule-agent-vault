---
title: "Entity / Content Headers"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:55:00+09:00
tags:
  - network
  - http
  - headers
  - entity
  - content
---

# Entity / Content Headers

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Content-Type/Length/Encoding/Language/Disposition/Range/Location |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

> body 에 부착되는 메타데이터. 요청 + 응답 양쪽 가능 (body 있을 때).

---

## 1. Content-Type

```http
Content-Type: application/json
Content-Type: application/json; charset=UTF-8
Content-Type: text/html; charset=UTF-8
Content-Type: image/jpeg
Content-Type: multipart/form-data; boundary=----X
```

### 구조
- **type/subtype** — MIME type
- **; parameters** — charset, boundary 등

### 주요 MIME types

#### text/*
- `text/plain`
- `text/html`
- `text/css`
- `text/javascript` (옛 — 모던은 `application/javascript` 또는 `text/javascript` 둘 다 가능)
- `text/csv`
- `text/markdown`
- `text/event-stream` — SSE

#### application/*
- `application/json` — JSON
- `application/xml`
- `application/x-www-form-urlencoded` — HTML 폼
- `application/octet-stream` — 임의 binary (기본)
- `application/pdf`
- `application/zip`
- `application/javascript`
- `application/wasm`
- `application/grpc` / `application/grpc+proto` — gRPC
- `application/problem+json` — RFC 7807 에러
- `application/ld+json` — JSON-LD
- `application/hal+json` — HAL Hypermedia

#### image/*
- `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/avif`, `image/svg+xml`

#### audio/* video/*
- `audio/mpeg`, `audio/ogg`, `audio/wav`
- `video/mp4`, `video/webm`, `video/ogg`

#### multipart/*
- `multipart/form-data` — 파일 업로드
- `multipart/mixed` — 여러 종류 결합
- `multipart/byteranges` — Range 응답
- `multipart/alternative` — 메일

### charset
```
Content-Type: text/html; charset=UTF-8
Content-Type: application/json; charset=UTF-8     (사실상 UTF-8 만)
```

→ UTF-8 모던 표준. JSON 은 RFC 8259 가 UTF-8 강제.

### 함정
- **MIME sniffing** — 브라우저가 Content-Type 무시하고 추측 → 보안 위험
- 방어: `X-Content-Type-Options: nosniff`
- 자세히 → [[../security/x-content-type-options]]

---

## 2. Content-Length

```http
Content-Length: 1234
```

### 의미
- body 크기 (byte)
- chunked 면 X (`Transfer-Encoding: chunked` 와 동시 X)

### 함정
- 잘못된 값 → 응답 stuck 또는 잘림
- HTTP/2 — frame 이 처리, Content-Length 는 정보만
- **Request Smuggling** — Content-Length + Transfer-Encoding 동시 시 위험

---

## 3. Content-Encoding

```http
Content-Encoding: gzip
Content-Encoding: br
Content-Encoding: zstd
Content-Encoding: deflate
Content-Encoding: identity     (압축 없음, 기본)
```

### 의미
- 자원 자체의 인코딩 (압축)
- End-to-end

### Transfer-Encoding 과 차이

| 측면 | Content-Encoding | Transfer-Encoding |
| --- | --- | --- |
| 적용 | end-to-end | hop-by-hop |
| 의미 | 자원 자체 | 전송만 |
| ETag | 다름 (Content 가 다름) | 같음 |

자세히 → [[../performance/compression-encoding]], [[../performance/transfer-encoding]]

---

## 4. Content-Language

```http
Content-Language: ko
Content-Language: en-US, fr-CA      (여러 언어)
```

### 의미
- 자원의 언어
- Accept-Language 의 응답 측

### 형식
- BCP 47 (RFC 5646)
- `ko`, `ko-KR`, `en-US`, `zh-Hans-CN`

---

## 5. Content-Disposition

```http
Content-Disposition: inline
Content-Disposition: attachment
Content-Disposition: attachment; filename="report.pdf"
Content-Disposition: attachment; filename*=UTF-8''%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf
```

### 종류
- **inline** — 브라우저에 표시 (기본)
- **attachment** — 다운로드 강제

### filename / filename*
- `filename="..."` — ASCII 만
- `filename*=UTF-8''<percent-encoded>` — 유니코드 (RFC 5987)
- 둘 다 보내면 `filename*` 우선 (모던 브라우저)

### 함정
- **Path Traversal** — `filename="../../../etc/passwd"` 위험
- 사용자 입력 → filename 직접 X
- HTML escape 또는 sanitize

### multipart/form-data 의 일부

```http
Content-Type: multipart/form-data; boundary=----X

------X
Content-Disposition: form-data; name="username"

Alice
------X
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<binary>
------X--
```

---

## 6. Content-Range

```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-1999/5000
Content-Length: 1000
```

### 형식
- `bytes <start>-<end>/<total>`
- `bytes 1000-1999/5000` — 1000-1999, 전체 5000
- `bytes */5000` — 416 응답 (잘못된 Range)

자세히 → [[../performance/range-requests]]

---

## 7. Content-Location

```http
HTTP/1.1 200 OK
Content-Location: /api/users/123.json
```

### 의미
- 자원의 **실제 URL**
- Content Negotiation 시 — `/users/123` 요청 → `.json` / `.xml` 표현 반환

### vs Location
- **Location** — 다음 URL (Redirect / 생성됨)
- **Content-Location** — 현재 응답의 실제 URL

---

## 8. Content-Security-Policy

별도 카테고리 — 보안. 자세히 → [[../security/csp]]

---

## 9. Content-MD5 (deprecated)

```http
Content-MD5: Q2hlY2sgSW50ZWdyaXR5IQ==
```

- body 의 MD5 (base64)
- RFC 7231 deprecated — MD5 의 약함 + 효용 X
- 대체: ETag, S3 의 ETag, HTTPS 의 무결성

---

## 10. Content-Digest / Repr-Digest (RFC 9530)

```http
Repr-Digest: sha-256=:<base64>:
Content-Digest: sha-256=:<base64>:
```

- 모던 무결성 헤더 (2024)
- HTTP 헤더 / body 의 변조 감지
- AWS Signature 와 결합

---

## 11. Expires

```http
Expires: Wed, 14 May 2026 12:00:00 GMT
```

### 의미
- 자원의 만료 시간 (절대)
- 옛 캐싱 — Cache-Control max-age 가 우선

### 함정
- 시계 동기 (NTP) 의존
- Cache-Control 와 충돌 시 Cache-Control 우선

---

## 12. Last-Modified

[[response-headers#9. Last-Modified]] / [[../caching/last-modified]]

---

## 13. ETag

[[response-headers#8. ETag]] / [[../caching/etag-conditional]]

---

## 14. 전체 예 — JSON 응답

```http
HTTP/1.1 200 OK
Date: Tue, 13 May 2026 12:00:00 GMT
Server: nginx
Content-Type: application/json; charset=UTF-8
Content-Length: 134
Content-Encoding: gzip
Content-Language: ko-KR
Cache-Control: max-age=300
ETag: "v1.0-abc"
Last-Modified: Tue, 13 May 2026 10:00:00 GMT
Vary: Accept-Encoding, Accept-Language

<gzip-encoded 134 bytes>
```

압축 전 JSON:
```json
{"id":123,"name":"Alice","email":"alice@example.com"}
```

---

## 15. 함정

### 함정 1 — Content-Type 누락
- 브라우저 MIME sniffing (불안)
- API: 기본 `application/octet-stream` → 클라 파싱 실패

### 함정 2 — Content-Length 잘못
일부 잘림 / stuck. 라이브러리에 맡기는 게 안전.

### 함정 3 — Content-Encoding 의 ETag
gzip / br 시 ETag 다름. **strong ETag 는 인코딩 별로**.

### 함정 4 — Content-Disposition 의 filename
Path Traversal / 한글 — `filename*=UTF-8''` 사용.

### 함정 5 — multipart boundary
양쪽 일치 — 라이브러리 자동.

### 함정 6 — Expires 와 Cache-Control 모순
Cache-Control 우선이지만 일관 X 면 혼동.

### 함정 7 — charset 누락
한글 깨짐 (Latin-1 으로 해석). UTF-8 명시.

---

## 16. 학습 자료

- **RFC 9110** Section 8
- IANA MIME Types Registry — https://www.iana.org/assignments/media-types/
- **RFC 5987** — filename*
- **RFC 9530** — Content-Digest

---

## 17. 관련

- [[headers]] — Headers hub
- [[content-negotiation]] — Accept-* / Vary
- [[../performance/compression-encoding]] — gzip/br
- [[../performance/range-requests]] — Content-Range
- [[../security/x-content-type-options]] — MIME sniffing
