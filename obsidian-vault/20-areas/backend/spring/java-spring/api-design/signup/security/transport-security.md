---
title: "전송 보안 — HTTPS / HSTS / CORS / CSP / SecureHeaders / WAF"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - https
  - hsts
  - csp
---

# 전송 보안 — HTTPS / HSTS / CORS / CSP / SecureHeaders / WAF

**[[security|↑ security hub]]**

> 네트워크 / HTTP 계층의 방어. 잘못 설정하면 **도청 / MITM / cross-origin 사고**.

---

## 1. 본 vault 결정

```yaml
transport:
  https-only: true                        # HTTP → 301 redirect
  hsts: max-age=31536000; includeSubDomains
  tls-version: 1.2+
  cors: explicit-whitelist
  csp: enabled (strict)
  secure-headers: enabled
  waf: aws-waf or cloudflare
```

---

## 2. HTTPS 강제

### 2.1 왜 필수

- HTTP = 평문 전송 → 도청 가능 (공용 WiFi).
- JWT / cookie 가 평문 노출 = 즉시 도용.
- 한국 PIPC / GDPR 의 안전성 확보 조치 의무.

### 2.2 구현

**서버 사이드 (ELB / ALB)**
```yaml
# ALB Listener
- Port: 443 (HTTPS, ACM 인증서)
- Port: 80 (HTTP → 301 redirect to HTTPS)
```

**Spring Boot 사이드**
```yaml
server:
  forward-headers-strategy: framework        # ELB 의 X-Forwarded-Proto 신뢰
```

```java
// 옵션 — Spring Security 가 HTTPS 강제
http.requiresChannel(channel -> channel
    .anyRequest().requiresSecure()
);
```

### 2.3 안 하면 무슨 문제

- 공격자가 공용 WiFi 에서 도청 → JWT 도용.
- 사이트 자체 평판 ↓ (Chrome 의 "안전하지 않음" 표시).

---

## 3. HSTS (HTTP Strict Transport Security)

### 3.1 무엇

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

→ "이 도메인은 HTTPS 만, 1년간 (31536000초) 기억해".

### 3.2 왜 필요

- HTTPS redirect 만 → 첫 HTTP 요청 시 MITM 가능 (downgrade attack).
- HSTS = browser 가 HTTPS 만 사용 강제.

### 3.3 구현

```java
http.headers(headers -> headers
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000)
    )
);
```

### 3.4 preload list

```
preload directive
```

- HSTS preload list (Chrome / Firefox / Edge 모두 사용).
- 첫 방문에도 HTTPS 강제.
- 신청: https://hstspreload.org

**언제**
- 도메인 전체가 HTTPS 안정 후 (rollback 어려움).
- 1년 max-age 검증 후 preload 신청.

---

## 4. TLS 버전

### 4.1 1.2+ 만

- TLS 1.0 / 1.1 = 옛 cipher (RC4, 3DES) 취약.
- TLS 1.3 = 권장 (handshake 빠름, 강한 cipher).

### 4.2 ALB / Cloudfront 설정

```
TLS Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
```

→ TLS 1.2 + 1.3 만 허용.

### 4.3 cipher 선택

- ECDHE-* (forward secrecy).
- AES-GCM (auth + encrypt).
- ChaCha20-Poly1305 (모바일 친화).

---

## 5. CORS — 화이트리스트

자세히: [[authentication-authorization#6 CORS]].

**핵심**
- `*` 절대 X (with credentials).
- 화이트리스트 명시.
- preflight 캐시 (max-age).

---

## 6. CSP (Content Security Policy)

### 6.1 무엇

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self'; img-src * data:; frame-ancestors 'none';
```

→ "스크립트 / style / 이미지를 어떤 origin 에서 로드 허용".

### 6.2 왜 필요

- XSS 발생 시 mitigation — 악의적 JS 가 외부 도메인 로드 X.
- iframe 차단 (clickjacking 방어).

### 6.3 구현

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; "
            + "script-src 'self'; "
            + "style-src 'self' 'unsafe-inline'; "
            + "img-src 'self' data: https:; "
            + "frame-ancestors 'none'; "
            + "form-action 'self'")
    )
);
```

### 6.4 보고 모드 (report-only)

```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

- 차단 안 함, 위반 시 endpoint 로 report.
- 운영 중 정책 적용 전에 테스트.

---

## 7. SecureHeaders

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
X-XSS-Protection: 1; mode=block      (옛 browser)
```

### 7.1 각 헤더의 "왜"

**X-Content-Type-Options: nosniff**
- browser 가 Content-Type 무시 X.
- 옛 IE 의 MIME sniffing 보안 사고 방어.

**X-Frame-Options: DENY**
- 우리 페이지가 iframe 으로 embed 안 됨.
- Clickjacking 방어.
- 단 CSP `frame-ancestors` 가 더 강력.

**Referrer-Policy**
- 다른 사이트로 navigate 시 Referer 헤더에 우리 URL 노출 정도.
- `strict-origin-when-cross-origin` = same-origin 만 path 포함, cross 는 origin 만.

**Permissions-Policy**
- iframe / API 가 사용 가능한 browser feature 제한.
- 카메라 / 마이크 / 위치 등 명시.

### 7.2 Spring Boot 구현

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .contentTypeOptions(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
    .permissionsPolicy(pp -> pp.policy("geolocation=(), microphone=(), camera=()"))
);
```

---

## 8. WAF (Web Application Firewall)

### 8.1 무엇

- HTTP 요청을 application 도달 전에 검사 / 차단.
- DDoS / SQL injection / XSS / 일반 공격 패턴 차단.

### 8.2 옵션

| 서비스 | 비용 (월) | 통합 |
| --- | --- | --- |
| **AWS WAF** | $5 + per rule + per req | AWS ALB / CloudFront |
| **Cloudflare WAF** | 무료~$20 | 글로벌 |
| **AWS Shield Standard** | 무료 | DDoS 기본 |
| **AWS Shield Advanced** | $3000/월 | 강한 DDoS |

### 8.3 본 vault 권장

- AWS WAF (ALB / CloudFront 통합).
- Rate-based rule (IP 당 분당 N 요청).
- Managed rule (OWASP Top 10).

### 8.4 함정

- WAF 가 정상 요청 false-positive 차단 → CS 폭주.
- 운영 모드 시작 = monitoring 모드 (block 안 함) → 안정 후 활성.

---

## 9. 인증서 (TLS / SSL)

### 9.1 ACM (AWS Certificate Manager)

- 무료 + 자동 갱신.
- ALB / CloudFront 통합.

### 9.2 Let's Encrypt

- 무료 + 90일 갱신.
- 자체 서버에 직접 발급.

### 9.3 EV (Extended Validation)

- 비싸지만 browser 의 "회사명" 표시.
- 금융 / 은행만.

---

## 10. Cookie 보안

```java
// refresh token cookie
ResponseCookie cookie = ResponseCookie.from("refresh", refreshToken)
    .httpOnly(true)               // JS 접근 X
    .secure(true)                  // HTTPS 만
    .sameSite("Strict")            // cross-site 요청에 X
    .path("/auth")                 // refresh endpoint 만
    .maxAge(Duration.ofDays(14))
    .build();
```

**각 옵션의 "왜"**

| 옵션 | 왜 |
| --- | --- |
| `httpOnly` | XSS 발생해도 JS 가 cookie 접근 X |
| `secure` | HTTPS 만 전송 — HTTP 시 cookie 안 보냄 |
| `sameSite=Strict` | cross-site 요청에 cookie 안 포함 (CSRF 방어) |
| `path=/auth` | 특정 path 에만 — 다른 endpoint 에 노출 X |

**SameSite 옵션**
- `Strict` — 가장 엄격 (다른 사이트에서 navigate 도 X).
- `Lax` — navigate (GET) 는 OK, POST 는 X. 보통 default.
- `None` — 모두 OK + Secure 필수. cross-site 가 필요한 경우만.

본 vault: refresh = `Strict` (cross-site 요청 불필요).

---

## 11. 함정 모음

### 함정 1 — HTTPS 만 (HSTS 없음)
첫 HTTP 요청 시 MITM downgrade.
→ HSTS + preload.

### 함정 2 — HSTS max-age 짧음 (1일)
거의 무효.
→ 1년+.

### 함정 3 — TLS 1.0 / 1.1 허용
취약 cipher.
→ 1.2+ 만.

### 함정 4 — CORS `*` + credentials
사양 위반.
→ 화이트리스트.

### 함정 5 — CSP 없음
XSS 발생 시 무방비.
→ default-src 'self'.

### 함정 6 — `'unsafe-inline'` script-src
인라인 script 허용 = XSS 부분 무력화.
→ nonce / hash 사용.

### 함정 7 — X-Frame-Options 없음
clickjacking 가능.
→ DENY + CSP frame-ancestors.

### 함정 8 — Cookie 의 SameSite 없음
default 가 browser 마다 다름. CSRF 위험.
→ 명시 (Strict / Lax).

### 함정 9 — Cookie 의 secure 누락
HTTP 시 cookie 노출.
→ secure: true.

### 함정 10 — WAF active mode 즉시 활성
false-positive 폭증.
→ monitoring → 안정 후 active.

### 함정 11 — 인증서 만료 안 갱신
사이트 다운.
→ 자동 갱신 (ACM / certbot) + 알람.

### 함정 12 — forward-headers-strategy 미설정
X-Forwarded-Proto 무시 → HTTPS 인데 redirect loop.
→ Spring `forward-headers-strategy: framework`.

---

## 12. 다른 컨텍스트

### 12.1 글로벌

```yaml
transport:
  https-only: true
  cdn: cloudfront
  waf: cloudflare + aws-waf
  region-failover: enabled
```

### 12.2 모놀리식 / 작은 규모

```yaml
transport:
  https-only: true
  hsts: 1y
  tls: 1.2+
  waf: optional (cloudflare 무료)
```

### 12.3 금융 / 의료

```yaml
transport:
  ev-certificate: required
  tls: 1.3 only
  hsts: preload
  waf: aws-waf + advanced shield
```

---

## 13. 관련

- [[security|↑ security hub]]
- [[authentication-authorization]] — CORS / SecurityConfig
- [[../design-decisions/token-model]] — cookie vs header
- [[attack-defense]] — XSS / CSRF
- 외부 — OWASP Secure Headers, hstspreload.org, AWS WAF
