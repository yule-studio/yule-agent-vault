---
title: "X-Frame-Options — Clickjacking 방어"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:15:00+09:00
tags:
  - network
  - http
  - security
  - clickjacking
---

# X-Frame-Options — Clickjacking 방어

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DENY / SAMEORIGIN / 모던 CSP frame-ancestors |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

페이지가 `<iframe>` 으로 임베드 되는 것을 차단. **Clickjacking** 공격 방어.
RFC 7034 (2013) — 비공식.

---

## 2. Clickjacking 공격

```html
<!-- 공격자의 evil.com -->
<style>
  iframe {
    width: 100%; height: 100%;
    opacity: 0.001;     /* 거의 투명 */
    position: absolute; top: 0;
  }
  .fake-button {
    position: absolute; top: 100px; left: 100px;
  }
</style>

<div class="fake-button">Click for free prize!</div>
<iframe src="https://bank.com/transfer-page"></iframe>

<!-- 사용자가 "free prize" 클릭 → 실제로 투명 bank.com 의 송금 버튼 클릭 -->
```

→ 사용자가 의도 X 행동.

---

## 3. 헤더 값

### DENY
```http
X-Frame-Options: DENY
```

- **어떤 사이트도** iframe 으로 임베드 불가
- 자기 사이트도 X
- 가장 안전

### SAMEORIGIN
```http
X-Frame-Options: SAMEORIGIN
```

- **같은 origin** 만 iframe 허용
- example.com 의 페이지가 example.com 의 iframe 안 OK

### ALLOW-FROM (deprecated)
```http
X-Frame-Options: ALLOW-FROM https://partner.com
```

- 특정 origin 만 허용
- Chrome / Safari 미지원 — 사실상 사장
- → **CSP frame-ancestors** 사용

---

## 4. CSP frame-ancestors (모던 대체)

```http
Content-Security-Policy: frame-ancestors 'none';
Content-Security-Policy: frame-ancestors 'self';
Content-Security-Policy: frame-ancestors https://partner.com https://another.com;
```

### 장점
- 여러 origin 허용
- 정규식 / wildcard 가능
- 모든 모던 브라우저 지원

### X-Frame-Options vs frame-ancestors

| 측면 | X-Frame-Options | frame-ancestors |
| --- | --- | --- |
| 출시 | 옛 | 모던 |
| 여러 origin | ❌ (ALLOW-FROM 약함) | ✅ |
| Wildcard | ❌ | ✅ (`*.partner.com`) |
| 브라우저 | 모든 (옛 호환) | 모든 (모던) |

→ **둘 다 보내는 게 안전** (옛 브라우저 호환).

---

## 5. 권장 설정

### 가장 안전 (대부분 사이트)

```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

### 일부 임베드 허용

```http
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: frame-ancestors 'self'
```

### 특정 파트너 임베드

```http
Content-Security-Policy: frame-ancestors 'self' https://partner.com
```

(X-Frame-Options 는 ALLOW-FROM 약하므로 생략)

---

## 6. 서버 설정

### Nginx
```nginx
add_header X-Frame-Options "DENY" always;
add_header Content-Security-Policy "frame-ancestors 'none'" always;
```

### Apache
```apache
Header always set X-Frame-Options "DENY"
Header always set Content-Security-Policy "frame-ancestors 'none'"
```

### Express
```javascript
app.use((req, res, next) => {
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('Content-Security-Policy', "frame-ancestors 'none'");
    next();
});

// 또는 helmet
const helmet = require('helmet');
app.use(helmet.frameguard({action: 'deny'}));
```

---

## 7. Clickjacking 외 방어

### 7.1 SameSite Cookie

```http
Set-Cookie: sessionid=abc; SameSite=Strict
```

- iframe 이 cross-site 면 cookie X — 위조 행동 못 함

### 7.2 CSRF Token

```html
<form>
  <input type="hidden" name="csrf_token" value="...">
  ...
</form>
```

→ iframe 이 token 모름 → 행동 X.

### 7.3 Sensitive action 의 추가 확인

- 송금 전 비밀번호 재입력
- OTP / 2FA

---

## 8. 함정

### 함정 1 — ALLOW-FROM 사용
Chrome / Safari 무시. frame-ancestors.

### 함정 2 — iframe 의 정당한 사용 차단
- 자기 사이트의 dashboard 의 widget
- partner 의 임베드
- → SAMEORIGIN 또는 frame-ancestors 명시

### 함정 3 — 둘 다 설정 안 함
- X-Frame-Options 없으면 모든 iframe 허용
- 명시 차단 / 허용.

### 함정 4 — 응용의 모든 페이지 같은 정책
- 로그인 / 송금은 DENY
- 공개 위젯 / 콘텐츠는 다를 수 있음
- → 응용 별 분리

### 함정 5 — CSP frame-ancestors 와 X-Frame-Options 충돌
- 한 응답에 다른 정책
- 브라우저: CSP 우선 (모던)
- 일관 유지

---

## 9. 학습 자료

- **RFC 7034** (X-Frame-Options)
- "Clickjacking" — OWASP
- CSP frame-ancestors — MDN

---

## 10. 관련

- [[security]] — Security hub
- [[csp]] — frame-ancestors
- [[../cors/same-origin-policy]]
- [[../cookies/cookie-security]] — SameSite
