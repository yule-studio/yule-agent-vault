---
title: "HSTS — HTTP Strict Transport Security"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:05:00+09:00
tags:
  - network
  - http
  - security
  - hsts
---

# HSTS — HTTP Strict Transport Security

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | max-age / includeSubDomains / preload |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

**HTTPS 강제** — 브라우저가 한 번 HSTS 헤더 받으면 그 도메인은 HTTPS 만 접근.
RFC 6797 (2012).

---

## 2. 헤더

```http
HTTP/1.1 200 OK
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

### 디렉티브
- **max-age=N** — N 초 동안 HSTS 적용 (필수)
- **includeSubDomains** — 모든 subdomain 도
- **preload** — 브라우저에 hard-coded list 등록

---

## 3. 동작

```
1. 사용자 → http://example.com
2. 브라우저 → 1 차: HTTP 요청 (취약)
3. 서버 → 301 → https://example.com + HSTS 헤더
4. 브라우저: HSTS 저장 (max-age 동안)
5. 이후 모든 http://example.com 요청:
   → 브라우저가 자동 https:// 로 변환
   → 절대 HTTP 안 보냄
```

→ 첫 방문 외 MITM 방어.

---

## 4. HSTS 의 한계 — TOFU

### Trust On First Use
- 첫 HTTP 요청은 여전히 취약
- 공격자가 첫 응답 변조 → HSTS 헤더 제거 → 영구 HTTP

### 해결 — HSTS Preload

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

브라우저에 hard-coded:
- Chrome / Firefox / Safari / Edge 모두 같은 list
- https://hstspreload.org/ 에 등록

→ 첫 방문도 HTTPS — TOFU 문제 해결.

---

## 5. Preload 의 영구성 — 신중

### 등록 조건 (hstspreload.org)
- HTTPS 사이트
- `max-age >= 31536000` (1 년)
- `includeSubDomains` 필수
- `preload` directive 필수

### 영구
- 등록 후 제거 매우 어려움 (수개월)
- 모든 subdomain 도 HTTPS 강제
- 잘못 등록 시 큰 문제 (HTTPS 못 끄게 됨)

→ **확실히** 영구 HTTPS 정책 후 등록.

---

## 6. 권장 설정 단계

### Step 1 — 짧은 max-age (테스트)
```http
Strict-Transport-Security: max-age=300
```
- 5 분만
- 문제 시 빠르게 복구 가능

### Step 2 — 1 주
```http
Strict-Transport-Security: max-age=604800; includeSubDomains
```
- includeSubDomains 검증

### Step 3 — 6 개월
```http
Strict-Transport-Security: max-age=15552000; includeSubDomains
```

### Step 4 — 2 년 + preload
```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```
- hstspreload.org 등록

---

## 7. 서버 설정

### Nginx
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

### Apache
```apache
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
```

### Caddy
```
header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
```

### Cloudflare
- Dashboard — SSL/TLS → Edge Certificates → HSTS

---

## 8. 함정

### 함정 1 — 처음부터 큰 max-age + preload
HTTPS 못 끄게 됨. 점진 늘림.

### 함정 2 — includeSubDomains 미검증
일부 subdomain 이 HTTP 만 → 모두 깨짐.

### 함정 3 — HTTP 응답에 HSTS
RFC 위반 + 무시. HTTPS 응답에만.

### 함정 4 — preload 제거 시도
브라우저 list 에서 빼는 데 수개월 + 일부 사용자 영원히 적용.

### 함정 5 — Self-signed 인증서 + HSTS
인증서 오류 = 영구 차단 (uncircumventable).

### 함정 6 — 개발 환경에서 preload
localhost / 내부 — HSTS 적용 시 디버깅 어려움.

---

## 9. HSTS 와 HTTP/3

- 첫 응답에 Alt-Svc: h3 도 함께
- 다음부터 HTTPS + HTTP/3
- HSTS 가 HTTP/3 활용 보장

---

## 10. 학습 자료

- **RFC 6797** (HSTS)
- https://hstspreload.org/
- web.dev — HSTS 가이드

---

## 11. 관련

- [[security]] — Security hub
- [[../../tls-ssl/tls-ssl]] — HTTPS
- [[../../topics/topics]] — Zero Trust
