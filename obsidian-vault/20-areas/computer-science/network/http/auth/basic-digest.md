---
title: "HTTP Basic & Digest Auth"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:40:00+09:00
tags:
  - network
  - http
  - auth
  - basic
  - digest
---

# HTTP Basic & Digest Auth

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Basic / Digest 동작 + 한계 |

**[[auth|↑ Auth]]** · **[[../http|↑↑ HTTP]]**

---

# Part A. Basic Authentication

## A.1 한 줄 정의

가장 단순한 HTTP 인증 — `username:password` Base64. RFC 7617.

## A.2 흐름

```
1. Client → Server: 보호된 자원 요청
2. Server → Client: 401 Unauthorized
   WWW-Authenticate: Basic realm="API"
3. Client → Server: Authorization: Basic <Base64(user:pass)>
4. Server: Base64 decode → 검증 → 200 또는 401
```

## A.3 헤더

```http
Authorization: Basic dXNlcjpwYXNzd29yZA==

→ Base64 decode → "user:password"
```

### Base64 != 암호화
- 평문 — HTTPS 필수
- 누구나 decode 가능

## A.4 curl

```bash
curl -u user:pass https://api.example.com/

# 또는
curl -H "Authorization: Basic $(echo -n user:pass | base64)" https://...
```

## A.5 fetch (JS)

```javascript
const auth = btoa('user:password');
fetch('/api', {
    headers: {'Authorization': `Basic ${auth}`}
});
```

## A.6 한계

- 평문 (Base64 의 단순 인코딩)
- HTTPS 외에 사용 X
- 매 요청 전체 자격 송신
- 비밀번호 변경 어려움
- MFA / OTP 지원 X
- Logout 어려움 (브라우저 캐시)

## A.7 사용 사례 (제한적)

- **내부 도구 / 관리자 패널** (HTTPS + IP 제한)
- **간단한 API** (서버-to-서버)
- **Nginx / Apache** 의 기본 인증 (Basic Auth file)
- **개발 / 테스트** 환경

→ Production 외부 사용자 — OAuth / Bearer / mTLS.

## A.8 Nginx 설정

```nginx
location /admin/ {
    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

`.htpasswd` 생성:
```bash
htpasswd -c /etc/nginx/.htpasswd user1
```

---

# Part B. Digest Authentication

## B.1 한 줄 정의

Basic 의 평문 비밀번호 문제 해결 — **challenge-response 해시**. RFC 7616.
거의 사용 X (옛 / 일부 IoT).

## B.2 흐름

```
1. Client → Server: 요청
2. Server → Client: 401
   WWW-Authenticate: Digest realm="API",
                     nonce="abc...",
                     qop="auth",
                     algorithm=SHA-256
3. Client: 해시 계산
   HA1 = SHA256(user:realm:password)
   HA2 = SHA256(method:uri)
   response = SHA256(HA1:nonce:nc:cnonce:qop:HA2)
4. Client → Server:
   Authorization: Digest username="user",
                  realm="API",
                  nonce="abc...",
                  uri="/protected",
                  response="...",
                  qop="auth",
                  nc=00000001,
                  cnonce="..."
5. Server: 같은 계산 + 검증
```

## B.3 vs Basic

| 측면 | Basic | Digest |
| --- | --- | --- |
| 비밀번호 송신 | 평문 (Base64) | 해시 만 |
| HTTPS 필수 | ✅ | 약 (해시여도 비밀번호 추측 가능) |
| 복잡도 | 낮음 | 높음 |
| 사용 | 단순 도구 | 거의 X |

## B.4 한계

- 서버가 plain text 비밀번호 저장 (또는 HA1) 필요 — bcrypt 호환 X
- 알고리즘 옵션 적음 (MD5 deprecated, SHA-256 늦게 추가)
- 브라우저 / 서버 호환성 일관 X
- HTTPS 가 표준 — Digest 의 의의 ↓

## B.5 현재 사용

- 일부 IoT 장치 (Wi-Fi / IP 카메라)
- 옛 임베디드
- 거의 사장

→ 모던 — **Bearer / OAuth**.

---

## 6. 다른 옛 인증

### NTLM (Microsoft)
- Windows AD
- HTTP 위에서 Negotiate

### Kerberos
- 기업 SSO
- Authorization: Negotiate

### Token (옛)
```http
Authorization: Token abc123
```

- 표준 X — `Bearer` 가 표준

---

## 7. 학습 자료

- **RFC 7617** (Basic)
- **RFC 7616** (Digest)
- MDN Authentication
- OWASP Authentication

---

## 8. 관련

- [[auth]] — Auth hub
- [[bearer-token]] — 모던 대체
- [[oauth2-flow]] — 위임 인증
