---
title: "Cookie Attributes — 모든 속성"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:45:00+09:00
tags:
  - network
  - http
  - cookies
  - same-site
---

# Cookie Attributes — 모든 속성

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Domain/Path/Expires/Max-Age/Secure/HttpOnly/SameSite/Partitioned/__Host-/__Secure- |

**[[cookies|↑ Cookies]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 전체 예

```http
Set-Cookie: sessionid=abc123;
            Domain=.example.com;
            Path=/;
            Expires=Wed, 14 May 2026 12:00:00 GMT;
            Max-Age=86400;
            Secure;
            HttpOnly;
            SameSite=Strict;
            Partitioned
```

---

## 2. Domain

```http
Set-Cookie: id=1; Domain=example.com
```

### 의미
- 쿠키가 적용될 도메인
- 명시 안 하면 **현재 도메인만** (subdomain X)
- 명시 시 — 그 도메인 + subdomain 모두

### 예
```
Set-Cookie: id=1                       → example.com 만
Set-Cookie: id=1; Domain=example.com   → example.com, sub.example.com
Set-Cookie: id=1; Domain=.example.com  → 위와 동일 (역사적 leading .)
```

### 한계
- 다른 도메인 설정 X — `Set-Cookie: id=1; Domain=evil.com` 거부

### Public Suffix List (PSL)
- `Domain=.co.uk` 처럼 too-broad 거부
- TLD / 공공 서브도메인 차단
- Mozilla 가 관리 — publicsuffix.org

---

## 3. Path

```http
Set-Cookie: id=1; Path=/api
```

### 의미
- 쿠키 송신 path prefix
- `/api/users` 에는 전송, `/admin` 에는 X

### 기본
- 현재 path 까지 — `/users/123` 응답이면 `Path=/users/`

### 함정
- "보안 메커니즘 X" — JS / XSS 가 다른 path 의 쿠키 접근 가능
- 신뢰 X — HttpOnly / SameSite 가 진짜 방어

---

## 4. Expires / Max-Age

```http
Set-Cookie: id=1; Expires=Wed, 14 May 2026 12:00:00 GMT
Set-Cookie: id=1; Max-Age=86400         (초)
```

### 차이
- **Expires** — 절대 시간 (HTTP date, GMT)
- **Max-Age** — 상대 시간 (초)

### 우선순위
- 둘 다 있으면 Max-Age 우선
- 둘 다 없으면 **Session cookie** (브라우저 닫으면 사라짐)

### 삭제
```http
Set-Cookie: id=; Max-Age=0
Set-Cookie: id=; Expires=Thu, 01 Jan 1970 00:00:00 GMT
```

→ 같은 name + 만료된 시간.

### 함정
- 클라 시계 — Expires 영향
- Max-Age 가 더 안전
- 너무 긴 Expires — 보안 위험 (도난 쿠키 오래 사용)

---

## 5. Secure

```http
Set-Cookie: id=1; Secure
```

### 의미
- **HTTPS 만** 송신
- HTTP 페이지엔 자동 X

### Cookie Prefix 와 결합
- `__Secure-` prefix 면 자동 Secure
- `__Host-` 도 Secure 필수

---

## 6. HttpOnly

```http
Set-Cookie: id=1; HttpOnly
```

### 의미
- **JS 접근 X** (`document.cookie` 안 보임)
- HTTP 요청엔 자동 동봉

### 방어
- **XSS** — 공격자가 JS 로 쿠키 훔치기 차단
- 세션 쿠키 의 표준

### 함정
- HttpOnly 만으로 XSS 방어 X — XSS 자체 방어 필요 (CSP, escape)
- HttpOnly 가 안 되면 XHR / fetch 가 쿠키 못 봄 (이건 OK)

---

## 7. SameSite (CSRF 방어)

```http
Set-Cookie: id=1; SameSite=Strict
Set-Cookie: id=1; SameSite=Lax
Set-Cookie: id=1; SameSite=None; Secure
```

### 의미

| 값 | 동작 |
| --- | --- |
| **Strict** | 같은 사이트만 — 다른 사이트에서 navigation 도 X |
| **Lax** | 같은 사이트 + Top-level navigation (link click OK) |
| **None** | 모든 cross-site (Secure 필수) |

### 기본값
- Chrome / Edge / Firefox 모던: **Lax** (2020+)
- 옛: None (취약)

### Strict 의 효과
```
사용자가 https://evil.com 에 방문
evil.com 의 <img src="https://bank.com/transfer?to=hacker">
→ 사용자 쿠키 첨부 X (SameSite=Strict)
→ CSRF 방어
```

### Lax 의 trade-off
- Strict 만큼 안전하지 X (top-level GET 은 쿠키 전송)
- 사용자 경험 ↑ (외부 링크 click 시 로그인 유지)

### None + Secure
- 진정한 cross-site cookie (광고 / 분석)
- Secure 강제 (HTTPS)
- Chrome 2020+ 적용

---

## 8. Partitioned (CHIPS)

```http
Set-Cookie: __Host-id=1; Secure; Path=/; SameSite=None; Partitioned
```

### 의미
- 3rd-party 사이트별 쿠키 분리
- top-level 사이트마다 별도 cookie jar

### 동기
- 3rd-party 차단 + 정당한 사용 (chat widget, payment) 가능
- Privacy Sandbox 의 일부

### 사용
- Embedded chat (Intercom, Drift)
- Payment iframe (Stripe Elements)
- 광고 / 분석은 다른 방법

---

## 9. Cookie Prefix — 보안 강화

### __Secure-

```http
Set-Cookie: __Secure-id=1; Secure; Path=/
```

- 자동 Secure 검증
- 브라우저가 강제

### __Host-

```http
Set-Cookie: __Host-id=1; Secure; Path=/
```

- Secure 필수
- **Domain 속성 X**
- **Path=/ 만**
- Subdomain 사이 분리 보장

→ XSS 도 subdomain takeover 도 방어.

### 강제 / 거부

```http
Set-Cookie: __Host-id=1; Domain=example.com       ← 거부 (Domain 있음)
Set-Cookie: __Secure-id=1; (no Secure)            ← 거부
```

---

## 10. 전체 권장 — 세션 쿠키

```http
Set-Cookie: __Host-session=abc;
            HttpOnly;
            Secure;
            SameSite=Strict;
            Path=/;
            Max-Age=3600
```

- `__Host-` prefix
- HttpOnly (XSS 방어)
- Secure (HTTPS)
- SameSite=Strict (CSRF)
- Path=/
- 짧은 Max-Age + Refresh token

---

## 11. 다른 도메인 / 사이트 정의

### Site vs Origin

```
Origin:   scheme + host + port  (예: https://app.example.com:443)
Site:     scheme + eTLD+1       (예: https://example.com)

→ Site 가 더 넓음
→ SameSite 는 Site 기준 (subdomain 도 same-site)
```

→ `app.example.com` 과 `api.example.com` 은 SameSite 기준 "same".

---

## 12. 함정

### 함정 1 — SameSite=None 의 Secure 누락
거부 — Chrome 80+.

### 함정 2 — Domain 의 leading dot
RFC 6265 — `.example.com` 의 dot 무시. `example.com` 과 동등.

### 함정 3 — Path 가 보안
보안 X — XSS 가 다른 path 의 쿠키 접근 가능.

### 함정 4 — Expires 의 시계
사용자 시계 변경 → 영향. Max-Age 권장.

### 함정 5 — HttpOnly + XSS 만 보면 안전
XSS 가 있으면 다른 공격 가능 (DOM, 페이지 조작). XSS 자체 방어.

### 함정 6 — SameSite=Lax 의 GET 누출
top-level GET 은 쿠키 전송 — 일부 CSRF 가능. 중요 작업은 POST + SameSite=Strict.

### 함정 7 — 3rd-party 차단의 영향
임베드된 iframe / 분석이 깨질 수 있음. Partitioned 또는 1st-party 통신.

---

## 13. 학습 자료

- **RFC 6265** / RFC 6265bis (modern)
- "SameSite cookies explained" — web.dev
- "Cookies Have Independent Partitioned State (CHIPS)" — Chrome
- MDN Set-Cookie

---

## 14. 관련

- [[cookies]] — Cookies hub
- [[cookie-security]] — XSS / CSRF 결합
- [[session-cookies]]
- [[../security/security]]
- [[../cors/cors-with-credentials]]
