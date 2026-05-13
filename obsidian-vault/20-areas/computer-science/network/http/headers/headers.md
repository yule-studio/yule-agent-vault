---
title: "HTTP 헤더 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:35:00+09:00
tags:
  - network
  - http
  - headers
---

# HTTP 헤더 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 분류 + 인덱스 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

요청 / 응답에 부착되는 **메타데이터** — Name: Value. RFC 9110 의 Semantics 가
모든 버전 공통.

---

## 2. 분류

### 2.1 의미 분류 (RFC 9110)

| 분류 | 어디 | 내용 |
| --- | --- | --- |
| **General** | 요청 + 응답 | 메시지 전체 (Date, Connection, Cache-Control) |
| **Request** | 요청만 | 클라 정보 (User-Agent, Host, Accept) |
| **Response** | 응답만 | 서버 정보 (Server, Location, WWW-Authenticate) |
| **Entity / Content** | body 가 있을 때 | Content-* (Type, Length, Encoding) |

→ 자세히:
- [[general-headers]]
- [[request-headers]]
- [[response-headers]]
- [[entity-headers]]
- [[content-negotiation]] — Accept-*, Vary
- [[custom-x-headers]] — X-Forwarded-For 등

---

## 3. 형식 (RFC 9110 Section 5)

### 일반 형식

```
[name]: [value]\r\n
```

### 규칙

- 이름 — case-insensitive (`Content-Type` = `content-type`)
- 값 — 공백 trim
- 콤마로 여러 값
- 줄 끝 `\r\n` (CRLF)
- 헤더 끝 — 빈 줄 (CRLF + CRLF)

### Multi-line (deprecated)

```
X-Custom: line1
 line2
```

→ RFC 7230 부터 폐기 — 보안 위험 (Request Smuggling).

---

## 4. 자주 쓰는 헤더 50

### General
- **Date** — `Tue, 13 May 2026 12:00:00 GMT`
- **Cache-Control** — 캐시 정책
- **Connection** — `keep-alive`, `close`, `Upgrade`
- **Via** — Proxy chain
- **Pragma** — 옛 캐시 (deprecated)
- **Trailer** — chunked 끝의 헤더 목록

### Request (가장 흔한)
- **Host** — 가상 호스팅 (HTTP/1.1 필수)
- **User-Agent** — 클라 식별
- **Accept** — 받을 수 있는 MIME
- **Accept-Language** — 선호 언어
- **Accept-Encoding** — 압축 (gzip/br)
- **Accept-Charset** — 거의 사용 X
- **Authorization** — 인증
- **Cookie** — 세션
- **Referer** — 출처 URL (오타 그대로)
- **Origin** — CORS / CSRF
- **If-Match / If-None-Match / If-Modified-Since / If-Unmodified-Since** — 조건부
- **Range** — 부분 요청
- **Expect** — 100-continue

### Response (가장 흔한)
- **Server** — 서버 식별 (보안상 숨김 권장)
- **Location** — Redirect / 새 자원
- **WWW-Authenticate** — 401 의 인증 방식
- **Proxy-Authenticate** — 407
- **Set-Cookie** — 쿠키 설정
- **Allow** — 허용 메서드 (405)
- **Retry-After** — 429/503 의 재시도 시점
- **ETag** — 자원 fingerprint
- **Last-Modified** — 자원 수정 시간
- **Vary** — 캐시 키
- **Strict-Transport-Security** — HSTS
- **Content-Security-Policy** — CSP
- **X-Frame-Options** — Clickjacking 방어

### Entity / Content
- **Content-Type** — MIME (`application/json; charset=UTF-8`)
- **Content-Length** — body 크기 (byte)
- **Content-Encoding** — `gzip`, `br`, `zstd`
- **Content-Language** — 언어
- **Content-Location** — 자원의 실제 URL
- **Content-Disposition** — `attachment; filename="file.pdf"`
- **Content-Range** — 부분 응답
- **Content-MD5** — body 의 MD5 (deprecated)

### CORS
- **Origin** (request)
- **Access-Control-Allow-Origin** (response)
- **Access-Control-Allow-Methods / Headers / Credentials / Max-Age**
- **Access-Control-Request-Method / Headers** (preflight)
- **Access-Control-Expose-Headers**

자세히 → [[../cors/cors]]

### Custom / X-
- **X-Forwarded-For** — 원본 클라 IP (Proxy / LB)
- **X-Forwarded-Proto** — 원본 프로토콜 (http / https)
- **X-Real-IP** — 원본 IP (대안)
- **X-Request-ID** — 추적 ID
- **X-API-Key** — API 키
- **X-CSRF-Token** — CSRF token

자세히 → [[custom-x-headers]]

---

## 5. 헤더 크기 한계

### 실제 한계 (서버 / 미들박스 별)

| 시스템 | 한계 |
| --- | --- |
| Nginx | 8 KB (1 헤더), 32 KB (전체) |
| Apache | 8 KB |
| AWS ALB | 16 KB (전체) |
| Cloudflare | 32 KB (전체) |
| Node.js (http) | 80 KB (전체) |

초과 시 → **431 Request Header Fields Too Large**.

### 흔한 원인
- Cookie 폭증 (광고 / 분석 추적기)
- 큰 referrer URL
- 다수 X- 헤더

---

## 6. 헤더 보안

### 6.1 민감 정보 누출 X
- **Authorization** — 로그에 남기지 X
- **Cookie** — `HttpOnly` + 마스킹
- **X-API-Key** — 로그 마스킹
- **Server** / **X-Powered-By** — 버전 노출 X (보안 스캔 표적)

### 6.2 Header Injection
- `\r\n` 삽입으로 헤더 분리 / 추가
- 응용 코드가 사용자 입력을 헤더로 — 검증 필수

### 6.3 보안 헤더 (응답)

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
```

자세히 → [[../security/security]]

---

## 7. 헤더 압축 (HTTP/2 HPACK / HTTP/3 QPACK)

- HTTP/1.1 — 평문 (반복으로 큰 오버헤드)
- HTTP/2 — HPACK (정적 + 동적 테이블 + Huffman)
- HTTP/3 — QPACK (HPACK + Stream 독립성)

→ 같은 헤더 반복 시 1 byte 도 가능.

자세히 → [[../versions/http-2#3.3 HPACK]]

---

## 8. tcpdump / curl 로 헤더 보기

```bash
# 요청 + 응답 헤더
curl -i https://example.com/

# Verbose (요청 + 응답)
curl -v https://example.com/

# 헤더만
curl -I https://example.com/

# 헤더 추가
curl -H "X-Custom: value" -H "Authorization: Bearer ..." https://...

# tcpdump (HTTP only, HTTPS 는 평문 안 보임)
sudo tcpdump -i en0 -A -s 0 'tcp port 80'
```

---

## 9. 함정

### 함정 1 — 대소문자
RFC: case-insensitive. **하지만 일부 라이브러리 / 응용은 case-sensitive**. 일관된 표기.

### 함정 2 — Comma-separated values
```
Accept: text/html, application/json
```
→ 콤마 = 여러 값. 공백 추가도 안전.

### 함정 3 — Header injection
사용자 입력 → 헤더 — `\r\n` 검증.

### 함정 4 — Custom 헤더의 미들박스 폐기
일부 Proxy 가 모르는 헤더 폐기. `X-` prefix 권장 (역사적).

### 함정 5 — 큰 헤더
431 — Cookie 정리.

### 함정 6 — Trailer
chunked 응답의 후행 헤더. 거의 사용 X.

### 함정 7 — 헤더 vs body 의 정보
신뢰성 — body 가 변조될 수 있음. Authorization / CSRF token 은 헤더 또는 cookie.

---

## 10. 학습 자료

- **RFC 9110** Section 5 (Headers)
- IANA HTTP Field Name Registry
- MDN HTTP Headers — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers

---

## 11. 관련

- [[../http]] — HTTP hub
- 카테고리별 — [[general-headers]], [[request-headers]], [[response-headers]], [[entity-headers]]
- [[content-negotiation]] — Accept-*
- [[custom-x-headers]] — X-Forwarded-For 등
- [[../security/security]] — 보안 헤더
