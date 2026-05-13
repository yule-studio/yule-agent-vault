---
title: "Request Headers — 요청 전용"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:45:00+09:00
tags:
  - network
  - http
  - headers
  - request
---

# Request Headers — 요청 전용

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Host / UA / Referer / Origin / Authorization / Cookie / If-* / Range |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

---

## 1. Host (HTTP/1.1 필수)

```http
Host: api.example.com:443
Host: example.com
```

### 의미
- 가상 호스팅 — 한 IP 의 여러 도메인
- HTTP/1.1 부터 필수
- HTTP/2 — `:authority` pseudo-header 가 대체

### 함정
- Host header injection — `Host: evil.com` 으로 응용 우회
- 응용 / 라우터의 Host 검증

### Nginx

```nginx
server {
    server_name api.example.com;
    if ($host !~* ^(api\.example\.com|www\.api\.example\.com)$) {
        return 444;   # 잘못된 Host → 차단
    }
}
```

---

## 2. User-Agent

```http
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/15.0 Safari/605.1.15
User-Agent: curl/7.68.0
User-Agent: Googlebot/2.1
```

### 의미
- 클라이언트 식별 (브라우저 / OS / 봇)
- 응용이 사용자 경험 조정

### 형식 (역사적)
- 1995 Netscape 부터 시작 — `Mozilla/...` prefix
- 호환성을 위해 모든 브라우저가 "Mozilla" 로 시작
- 매우 길고 모호 — 파싱 어려움

### 모던 — User-Agent Client Hints (RFC 8942)

```http
Sec-CH-UA: "Chromium";v="120", "Not?A_Brand";v="99"
Sec-CH-UA-Platform: "macOS"
Sec-CH-UA-Mobile: ?0
```

- UA 의 정보를 작은 hint 로 분리
- 프라이버시 — 필요할 때만 보내기
- Chrome 의 UA 축소 정책 (2022+)

### 함정
- UA Spoofing — 신뢰 X
- 브라우저 식별은 feature detection (JS)
- 봇 차단은 UA 외 (IP / 행동 분석)

---

## 3. Referer

```http
Referer: https://google.com/?q=cats
```

### 의미
- 클라이언트가 어디서 왔는지 (이전 페이지 URL)
- **철자 "Referrer" 의 오타가 표준화** (역사적)

### 사용
- 분석 (referrer tracking)
- CSRF 방어 (제한적)
- 핫링크 차단

### 보안 / 프라이버시
- URL 의 query string 노출
- `Referrer-Policy` 헤더로 제어 (응답)

자세히 → [[../security/referrer-policy]]

```http
Referrer-Policy: strict-origin-when-cross-origin
```

---

## 4. Origin

```http
Origin: https://app.example.com
```

### 의미
- 요청의 출처 (scheme + host + port)
- Referer 와 비슷하지만 path 없음 — 더 안전

### 사용
- **CORS** — 브라우저가 cross-origin 요청에 자동 추가
- **CSRF 방어** — 서버가 Origin 검증
- POST / PUT / DELETE / PATCH 에 자동 (브라우저)

### Origin vs Referer

| 측면 | Origin | Referer |
| --- | --- | --- |
| 내용 | scheme + host + port | 전체 URL |
| 프라이버시 | 좋음 | 약함 |
| CORS | 사용 | X |
| 모던 권장 | ✅ | △ |

자세히 → [[../cors/cors]]

---

## 5. Authorization

```http
Authorization: Basic dXNlcjpwYXNz
Authorization: Bearer eyJhbGc...
Authorization: Digest username="user", realm="...", nonce="...", uri="/api"
```

### Scheme

| Scheme | 의미 |
| --- | --- |
| **Basic** | Base64(user:pass) — HTTPS 필수 |
| **Bearer** | Token (JWT, OAuth) |
| **Digest** | Challenge-response, MD5/SHA |
| **AWS4-HMAC-SHA256** | AWS 서명 |
| **Negotiate** | Kerberos / NTLM |

자세히 → [[../auth/auth]]

### 보안
- 로그에 남기지 X — 마스킹
- HTTPS 외에서 보내지 X

---

## 6. Cookie

```http
Cookie: sessionid=abc123; user_id=42; theme=dark
```

### 형식
- `name=value; name=value;` (semicolon)
- 도메인 / Path 의 모든 쿠키 자동 동봉
- HttpOnly 도 자동 (JS X 인데 HTTP 요청엔 포함)

자세히 → [[../cookies/cookies]]

---

## 7. Accept / Accept-* (Content Negotiation)

```http
Accept: application/json, text/html;q=0.9, */*;q=0.5
Accept-Language: ko, en-US;q=0.8, en;q=0.5
Accept-Encoding: gzip, br, zstd
Accept-Charset: utf-8       (deprecated)
```

자세히 → [[content-negotiation]]

---

## 8. If-Match / If-None-Match / If-Modified-Since / If-Unmodified-Since

조건부 요청 — 캐시 / 동시성 제어.

```http
GET /api/users/123 HTTP/1.1
If-None-Match: "v1"
If-Modified-Since: Tue, 13 May 2026 10:00:00 GMT

→ 304 Not Modified 또는 200 OK
```

```http
PUT /api/users/123 HTTP/1.1
If-Match: "v1"

→ 200 OK 또는 412 Precondition Failed
```

자세히 → [[../caching/etag-conditional]], [[../caching/last-modified]]

### If-None-Match: *

```http
PUT /api/users/123 HTTP/1.1
If-None-Match: *
```

→ "존재하지 않을 때만 생성" — atomic create.

---

## 9. Range

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1000-2000
Range: bytes=0-499, 1000-1499      (multiple)
Range: bytes=-500                   (마지막 500)
Range: bytes=9500-                  (9500 부터 끝)
```

### 응답
- 200 OK (full) — Range 무시
- 206 Partial Content (부분)
- 416 Range Not Satisfiable (잘못된 range)

자세히 → [[../performance/range-requests]]

---

## 10. Expect

```http
Expect: 100-continue
```

→ 큰 body 전 100 Continue 받기.

자세히 → [[../status-codes/1xx-informational]]

---

## 11. Forwarded / X-Forwarded-*

```http
Forwarded: for=192.0.2.1; proto=https; host=example.com
X-Forwarded-For: 192.0.2.1, 198.51.100.5
X-Forwarded-Proto: https
X-Forwarded-Host: example.com
```

자세히 → [[custom-x-headers]]

---

## 12. DNT — Do Not Track (deprecated)

```http
DNT: 1
```

- 추적 거부 명시
- 응용 / 광고 측이 무시해 사실상 폐기
- Global Privacy Control (GPC) 가 후속

---

## 13. Sec-Fetch-* (모던, Fetch Metadata)

```http
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Sec-Fetch-User: ?1
```

### 의미
- 브라우저가 자동 추가 (사용자 입력 X)
- 서버가 요청의 의도 추론 → CSRF 방어 강화

### 값
- **Sec-Fetch-Site**: same-origin, same-site, cross-site, none
- **Sec-Fetch-Mode**: cors, navigate, no-cors, same-origin, websocket
- **Sec-Fetch-Dest**: document, image, script, style, audio, video, font, empty
- **Sec-Fetch-User**: `?1` (사용자가 직접 시작) / 없음

자세히 → [[../security/security]]

---

## 14. Content-* (Request Body — Entity 의 일부)

```http
POST /api/users HTTP/1.1
Content-Type: application/json
Content-Length: 56
Content-Encoding: gzip
Content-Language: ko
```

자세히 → [[entity-headers]]

---

## 15. 함정

### 함정 1 — Host 의 case
대소문자 무관 — `host: example.com` 도 OK.

### 함정 2 — UA 신뢰 X
Spoofing 쉬움. 신뢰가 필요한 결정은 X.

### 함정 3 — Referer 의 누락
- HTTPS → HTTP 이동 시 자동 제거 (보안)
- 사용자 설정으로 무력화
- 분석 / 차단 의존 X

### 함정 4 — Origin 의 신뢰
브라우저는 자동 추가 — Spoofing X. 단 non-browser 클라 (curl) 는 임의.

### 함정 5 — Authorization vs WWW-Authenticate
요청 / 응답 짝.

### 함정 6 — Forwarded vs X-Forwarded-For
RFC 7239 의 `Forwarded` 가 표준, `X-` 가 historical. 둘 다 함정.

### 함정 7 — Sec-Fetch-* 미지원 브라우저
Safari 옛 / Edge 옛 / 일부 모바일 — 헤더 없음. 폴백 필요.

---

## 16. 학습 자료

- **RFC 9110** Section 10
- **RFC 8941** — Structured Headers
- **RFC 8942** — UA Client Hints
- MDN Request headers

---

## 17. 관련

- [[headers]] — Headers hub
- [[response-headers]] — 짝
- [[content-negotiation]] — Accept-*
- [[../caching/etag-conditional]] — If-*
- [[../performance/range-requests]] — Range
- [[../auth/auth]] — Authorization
- [[../cors/cors]] — Origin
