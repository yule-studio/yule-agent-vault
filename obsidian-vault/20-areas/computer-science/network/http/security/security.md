---
title: "HTTP 보안 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:00:00+09:00
tags:
  - network
  - http
  - security
---

# HTTP 보안 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | HTTP 보안 헤더 + 위협 정리 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

> 응용 보안 / 인증 일반 은 [[../../../security-theory/security-theory|↗ security-theory]].
> 이 폴더는 **HTTP 의 보안 헤더 / 패턴**.

---

## 1. HTTP 의 보안 헤더 (응답)

```http
HTTP/1.1 200 OK
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
```

→ 모던 사이트의 기본 보안 헤더.

---

## 2. 위협 / 방어 매핑

| 위협 | 방어 헤더 / 패턴 | 노트 |
| --- | --- | --- |
| **MITM** | HTTPS + HSTS | [[hsts]] |
| **XSS** | CSP, HttpOnly Cookie | [[csp]] |
| **Clickjacking** | X-Frame-Options, CSP frame-ancestors | [[x-frame-options]] |
| **MIME sniffing** | X-Content-Type-Options: nosniff | [[x-content-type-options]] |
| **Referer 누출** | Referrer-Policy | [[referrer-policy]] |
| **Browser feature 남용** | Permissions-Policy | [[permissions-policy]] |
| **CSRF** | SameSite Cookie + CSRF Token + Origin 검증 | [[../cookies/cookie-security]] |
| **Sub-resource 침해** | SRI (Subresource Integrity) | — |
| **Spectre / Side-channel** | COOP / COEP / CORP | — |

---

## 3. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[hsts]] | Strict-Transport-Security |
| [[csp]] | Content-Security-Policy |
| [[x-frame-options]] | Clickjacking 방어 |
| [[x-content-type-options]] | MIME sniffing 방어 |
| [[referrer-policy]] | Referer 헤더 제어 |
| [[permissions-policy]] | Feature-Policy 후속 |

---

## 4. SRI — Subresource Integrity

```html
<script src="https://cdn.com/lib.js"
        integrity="sha384-abc..."
        crossorigin="anonymous"></script>
```

### 의미
- CDN 의 자원 무결성 검증
- 해시 일치 안 하면 폐기
- CDN 해킹 / MITM 방어

### 생성
```bash
shasum -a 384 lib.js | xxd -r -p | base64
```

---

## 5. Mixed Content

```
HTTPS 페이지가 HTTP 자원 로드
→ 브라우저: 차단 또는 경고
```

### 종류
- **Passive (image / video)** — 옛 브라우저 OK, 모던 차단
- **Active (script / iframe)** — 차단 강

### 방어
- 모든 자원 HTTPS
- `<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">`

---

## 6. Cross-Origin Isolation (Spectre 이후)

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
```

### COOP — Cross-Origin Opener Policy
- 다른 origin 이 `window.opener` 로 자기 봄 차단

### COEP — Cross-Origin Embedder Policy
- 임베드 자원 모두 CORP 또는 CORS 필수

### CORP — Cross-Origin Resource Policy
- 다른 origin 의 임베드 차단

### 효과
- `crossOriginIsolated = true` 가 됨
- SharedArrayBuffer / 정밀 timer 사용 가능
- Spectre 류 사이드 채널 방어

---

## 7. Cookie 보안

자세히 → [[../cookies/cookie-security]]

```http
Set-Cookie: __Host-session=abc;
            HttpOnly;
            Secure;
            SameSite=Strict;
            Max-Age=3600;
            Path=/
```

---

## 8. 인증 / 인가

자세히 → [[../auth/auth]]

- Basic / Digest
- Bearer (JWT / OAuth)
- API Key
- mTLS

---

## 9. Rate Limiting

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1715587200
```

### 알고리즘
- **Token Bucket** — 토큰 충전 + 소비
- **Leaky Bucket** — 큐 + 일정 속도
- **Fixed Window** — 시간 창
- **Sliding Window** — 정확한 시간 창

### 구현
- Redis (분산)
- Nginx `limit_req_zone`
- Cloudflare / AWS WAF

---

## 10. WAF — Web Application Firewall

### 패턴 매칭
- SQL Injection, XSS, RCE
- OWASP CRS (Core Rule Set)
- ModSecurity (Open) / Cloudflare WAF / AWS WAF

### Behavior 기반
- 비정상 트래픽 (스캐너 / 봇)
- IP 차단 / Captcha

---

## 11. HTTPS 권장 헤더 — 모범 세트

```http
# HTTPS 강제
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload

# XSS / Injection 방어
Content-Security-Policy: default-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'

# Clickjacking (CSP 의 frame-ancestors 가 대체)
X-Frame-Options: DENY

# MIME sniffing
X-Content-Type-Options: nosniff

# Referer 제한
Referrer-Policy: strict-origin-when-cross-origin

# 기능 제한
Permissions-Policy: geolocation=(), microphone=(), camera=()

# Cross-origin isolation (필요 시)
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
```

---

## 12. 도구 / 점검

### securityheaders.com
- 헤더 자동 점검 + 등급

### Mozilla Observatory
- https://observatory.mozilla.org/
- 종합 점수

### OWASP ZAP
- 보안 스캐너 (오픈)

### Burp Suite
- Pentest 표준

---

## 13. 함정

### 함정 1 — HTTPS 만 설치
헤더 없으면 다른 공격 가능. 위 모범 세트.

### 함정 2 — CSP 너무 관대
`unsafe-inline`, `unsafe-eval` — 거의 보호 X.

### 함정 3 — HSTS preload 의 영구
preload 등록 후 HTTPS 못 끄게 됨. 신중.

### 함정 4 — Cookie 보안 누락
HttpOnly + Secure + SameSite=Strict.

### 함정 5 — Mixed Content
HTTPS 페이지 + HTTP 자원 → 브라우저 차단.

### 함정 6 — Rate Limit 없음
무차별 요청 / scraping / DDoS.

---

## 14. 학습 자료

- **OWASP** — owasp.org
- **OWASP Cheat Sheet Series**
- "Security Headers Quick Reference" — Mozilla
- web.dev — Security & Identity

---

## 15. 관련

- [[../http]] — HTTP hub
- [[hsts]] / [[csp]] / [[x-frame-options]] / [[x-content-type-options]] / [[referrer-policy]] / [[permissions-policy]]
- [[../cookies/cookie-security]]
- [[../auth/auth]]
- [[../cors/cors]]
- [[../../../security-theory/security-theory]]
