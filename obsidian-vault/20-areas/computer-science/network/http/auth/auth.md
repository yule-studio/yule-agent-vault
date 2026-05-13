---
title: "HTTP 인증 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:35:00+09:00
tags:
  - network
  - http
  - auth
  - authentication
---

# HTTP 인증 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Basic / Digest / Bearer / OAuth / API Key / mTLS |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. Authorization 헤더

```http
Authorization: <scheme> <credentials>

예:
Authorization: Basic dXNlcjpwYXNz
Authorization: Bearer eyJhbGc...
Authorization: Digest username="user", realm="...", ...
```

---

## 2. 7 가지 인증 방식

| Scheme | 사용 | 보안 |
| --- | --- | --- |
| **Basic** | 단순 / 옛 | 약 (Base64 평문, HTTPS 필수) |
| **Digest** | 옛 | 중 (challenge-response) |
| **Bearer** | OAuth / JWT | HTTPS + 토큰 보안 |
| **API Key** | 외부 API | HTTPS + key 관리 |
| **HMAC / Signature** | AWS, Stripe | 강 (요청 서명) |
| **OAuth 2.0** | 위임 | 모던 표준 |
| **mTLS** | 강력한 인증 | 매우 강 |

---

## 3. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[basic-digest]] | HTTP Basic / Digest |
| [[bearer-token]] | Authorization: Bearer (JWT) |
| [[oauth2-flow]] | OAuth 2.0 grant types |
| [[api-keys]] | API Key 패턴 |
| [[mtls-cert-auth]] | 클라이언트 인증서 |

---

## 4. 401 vs 403

| 측면 | 401 | 403 |
| --- | --- | --- |
| 의미 | 인증 X | 인증 OK, 인가 X |
| WWW-Authenticate | 필수 | 옵션 |
| 재시도 | 자격 시도 | 자격 다시 받아도 X |

자세히 → [[../status-codes/4xx-client-errors]]

---

## 5. WWW-Authenticate 헤더

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="API"
WWW-Authenticate: Bearer realm="API", error="invalid_token"
WWW-Authenticate: Digest realm="...", nonce="..."
```

→ 클라가 어떤 방식으로 인증할지 알림.

---

## 6. Session vs Token

| 측면 | Session Cookie | JWT (Bearer) |
| --- | --- | --- |
| 저장 | 서버 store | 토큰 자체 |
| 상태 | Stateful | Stateless |
| 무효화 | 즉시 | 만료까지 |
| 모바일 | 어려움 | 쉬움 |

자세히 → [[../cookies/jwt-vs-cookie]]

---

## 7. 현대 API 의 표준

### 7.1 외부 사용자 — OAuth 2.0
- Google / Facebook / GitHub 로그인
- API access (OAuth + OIDC)

### 7.2 Service-to-Service — Bearer Token / mTLS
- API Gateway 가 JWT 검증
- mTLS for sensitive

### 7.3 외부 개발자 — API Key
- Stripe / Twilio 등
- Rate limit + scope

### 7.4 내부 / Microservice — mTLS
- Service Mesh (Istio)
- Zero Trust

---

## 8. 면접 / 토픽

1. **401 vs 403** — 인증 vs 인가.
2. **JWT vs Session Cookie**.
3. **OAuth 2.0 의 grant types**.
4. **mTLS 와 사용 사례**.
5. **API Key 의 보안**.

---

## 9. 학습 자료

- **RFC 7235** (HTTP Authentication)
- **RFC 6750** (Bearer Token)
- **RFC 6749** (OAuth 2.0)
- OWASP Authentication Cheat Sheet

---

## 10. 관련

- [[../http]] — HTTP hub
- [[basic-digest]] / [[bearer-token]] / [[oauth2-flow]] / [[api-keys]] / [[mtls-cert-auth]]
- [[../cookies/session-cookies]]
- [[../security/security]]
- [[../../../security-theory/security-theory]]
