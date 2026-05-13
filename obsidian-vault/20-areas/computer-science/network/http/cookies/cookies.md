---
title: "쿠키 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:40:00+09:00
tags:
  - network
  - http
  - cookies
---

# 쿠키 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 쿠키 hub + 5 세부 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

서버가 클라이언트 브라우저에 **저장 + 자동 동봉** 시키는 작은 데이터. RFC 6265
(2011) → RFC 6265bis (모던). HTTP 의 **stateless** 한계 보완.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1994 | Lou Montulli (Netscape) — Cookie 발명 |
| 1997 | RFC 2109 — 첫 표준 |
| 2000 | RFC 2965 (옛) |
| 2011 | **RFC 6265** — 모던 표준 |
| 2016+ | SameSite 도입 (CSRF 방어) |
| 2020+ | 3rd-party Cookie 차단 (Safari ITP → Chrome 2024 계획) |
| 2024 | Partitioned (CHIPS), Privacy Sandbox |

---

## 3. 흐름

### 첫 응답 — Set-Cookie

```http
HTTP/1.1 200 OK
Set-Cookie: sessionid=abc123; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
```

### 후속 요청 — Cookie 자동

```http
GET /api/me HTTP/1.1
Cookie: sessionid=abc123
```

→ 브라우저가 자동 동봉 (응용 코드 X).

---

## 4. 쿠키 구조

```
name=value; attr1=v1; attr2=v2; ...

예:
sessionid=abc123; Domain=.example.com; Path=/; Expires=Wed, 14 May 2026 12:00:00 GMT; HttpOnly; Secure; SameSite=Strict
```

### name=value
- `name` — case-sensitive, 영문/숫자/일부 특수
- `value` — string (보통 URL encoded)
- 크기 한계 ~ 4096 byte / 쿠키 (browser 별)

### Attributes
자세히 → [[cookie-attributes]]

---

## 5. 종류

### 5.1 Session vs Persistent

| 종류 | 특징 |
| --- | --- |
| **Session cookie** | Expires / Max-Age 없음 — 브라우저 닫으면 사라짐 |
| **Persistent cookie** | Expires 또는 Max-Age 지정 — 명시 시간까지 |

### 5.2 First-party vs Third-party

| 종류 | 도메인 |
| --- | --- |
| **First-party** | 현재 페이지의 도메인 (example.com) |
| **Third-party** | 다른 도메인 (광고 / 분석) |

→ 모던: 3rd-party 차단 (privacy).

### 5.3 Secure vs Insecure

- **Secure** — HTTPS 만
- **Insecure** — HTTP 도 (옛)

### 5.4 HttpOnly vs Accessible

- **HttpOnly** — JS 접근 X
- 일반 — `document.cookie` 로 JS 접근

---

## 6. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[cookie-attributes]] | Domain / Path / Expires / Max-Age / Secure / HttpOnly / SameSite / Partitioned |
| [[cookie-security]] | XSS / CSRF / Session Hijacking 방어 |
| [[session-cookies]] | 서버 세션 / Session ID |
| [[jwt-vs-cookie]] | JWT 토큰 vs Cookie 비교 |

---

## 7. 한계

### 크기
- 한 쿠키 ~ 4 KB
- 한 도메인 ~ 50 cookie
- 모든 도메인 합 ~ 3000-4000

### 매 요청에 전송
- 같은 도메인의 모든 path 에 자동 (Path 매칭)
- 큰 쿠키 = 매 요청 오버헤드
- API 요청에도 — 비효율

### CORS
- 기본 — cross-origin 자동 X
- `credentials: include` (fetch) — 명시
- `Access-Control-Allow-Credentials` 필요

자세히 → [[../cors/cors-with-credentials]]

---

## 8. JS 접근

```javascript
// 읽기 (HttpOnly 제외)
document.cookie;
// "theme=dark; lang=ko"

// 쓰기
document.cookie = "theme=dark; path=/; max-age=86400; secure; samesite=strict";

// 삭제 (과거 만료)
document.cookie = "theme=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT";
```

### 한계
- HttpOnly 쿠키는 안 보임
- 한 줄에 모든 쿠키 — 파싱 필요
- 모던: **Cookie Store API** (Chrome / Edge)

```javascript
const cookies = await cookieStore.getAll();
await cookieStore.set({name: "theme", value: "dark", ...});
```

---

## 9. 모바일 / Native 앱

- WebView 안의 쿠키 — 브라우저와 분리
- iOS WKWebView — 격리
- Android WebView — 격리

→ 모바일 SDK 는 쿠키 대신 토큰 (JWT) + Keychain.

자세히 → [[jwt-vs-cookie]]

---

## 10. 미래 — Privacy Sandbox

### 3rd-party 차단 (Chrome 2024+)
- Safari ITP 2017+ 가 시작
- Firefox Total Cookie Protection 2022
- Chrome — 점진 차단 → 2024+ 계획

### 대체 — Partitioned (CHIPS)
- `Set-Cookie: ...; Partitioned`
- 첫 방문 사이트별 분리 cookie
- 광고 / 분석의 추적 방지

### Topics API / Protected Audience
- 광고 산업의 새 API
- 쿠키 없이 광고 매칭

---

## 11. 면접 / 토픽

1. **Cookie vs Session vs JWT**.
2. **HttpOnly / Secure / SameSite** — CSRF / XSS 방어.
3. **3rd-party cookie 의 미래**.
4. **CORS + Credentials**.
5. **Set-Cookie 흐름**.

---

## 12. 학습 자료

- **RFC 6265** / RFC 6265bis (모던 진행)
- MDN HTTP Cookies
- "SameSite cookies explained" — web.dev
- Privacy Sandbox — Google

---

## 13. 관련

- [[../http]] — HTTP hub
- [[cookie-attributes]] — 모든 속성
- [[cookie-security]] — XSS / CSRF
- [[session-cookies]] / [[jwt-vs-cookie]]
- [[../headers/request-headers]] — Cookie 헤더
- [[../headers/response-headers]] — Set-Cookie
- [[../security/security]]
- [[../cors/cors-with-credentials]]
