---
title: "Custom / X- Headers"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:05:00+09:00
tags:
  - network
  - http
  - headers
  - custom
  - x-forwarded-for
---

# Custom / X- Headers

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | X-Forwarded-* / X-Request-ID / X-API-Key / X- 정책 |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

---

## 1. X- prefix 정책

### 1.1 옛 (1990-2010)
- 비표준 헤더 = `X-` prefix
- `X-Forwarded-For`, `X-Powered-By`, `X-CSRF-Token` 등

### 1.2 RFC 6648 (2012)
- "X- prefix 사용 금지"
- 이유: 비표준이 표준 되면 이름 바꾸기 어려움 (`X-Forwarded-For` → `Forwarded` RFC 7239)
- 새 헤더는 prefix 없이

### 1.3 현실
- 기존 X- 헤더는 여전히 보편 (호환성)
- 새 헤더는 prefix 없이 (예: `Authorization`, `Strict-Transport-Security`)

---

## 2. X-Forwarded-* (LB / Proxy)

원본 클라 정보 — Proxy / LB 가 추가:

### 2.1 X-Forwarded-For (XFF)

```http
X-Forwarded-For: 192.0.2.1                      (직접)
X-Forwarded-For: 192.0.2.1, 198.51.100.5         (proxy chain)
```

- 가장 왼쪽 = 원본 클라
- 다음은 첫 proxy, 두 번째 proxy, ...
- 마지막은 직접 연결한 LB / proxy

### 2.2 X-Forwarded-Proto

```http
X-Forwarded-Proto: https
```

- LB 가 SSL 종료 후 백엔드에 HTTP — 원본 프로토콜 알림
- Spring / Django 등이 redirect / cookie Secure 결정에 사용

### 2.3 X-Forwarded-Host

```http
X-Forwarded-Host: www.example.com
```

- 원본 Host (LB 가 Host 변경 시)

### 2.4 X-Forwarded-Port

```http
X-Forwarded-Port: 443
```

### 2.5 X-Real-IP (Nginx)

```http
X-Real-IP: 192.0.2.1
```

- Nginx 의 단일 IP (XFF 의 chain 보다 단순)

---

## 3. Forwarded (RFC 7239)

X- 의 표준화 — 한 헤더에 모두:

```http
Forwarded: for=192.0.2.1; proto=https; host=example.com; by=203.0.113.43
Forwarded: for=192.0.2.1, for=198.51.100.17
```

### 매개변수
- **for** — 원본 클라
- **by** — 받은 proxy
- **proto** — http/https
- **host** — 원본 Host

### 호환
- X-Forwarded-* 가 여전히 더 많이 사용
- 모던 응용은 둘 다 처리

---

## 4. 신뢰 — IP Spoofing 방어

### 함정
```http
X-Forwarded-For: 1.2.3.4    ← 클라가 임의로 보낼 수 있음
```

→ 단순 신뢰 X. **신뢰 가능한 proxy 만 검증**.

### 패턴 — Trusted Proxy

```
Step 1: 신뢰 가능 proxy IP 화이트리스트
Step 2: 직접 연결한 IP 가 trusted 면 XFF 사용
Step 3: trusted 가 아니면 XFF 무시 (직접 IP 사용)
```

### Express 예

```javascript
app.set('trust proxy', '10.0.0.0/8, 192.168.0.0/16');
// 이제 req.ip 가 XFF 의 원본 IP (trusted 시)
```

### Nginx 예

```nginx
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.0.0/16;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

### Cloudflare
- `CF-Connecting-IP` — 원본 IP (XFF 보다 신뢰)
- Cloudflare 의 IP 만 신뢰

---

## 5. X-Request-ID — 분산 추적

```http
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

### 의미
- LB / Gateway / Application 에 동일 ID 부착
- 로그 추적 / 분산 trace
- 클라가 보내면 그대로, 없으면 서버 생성

### 변형 / 표준
- **X-Request-ID** — 비표준 보편
- **X-Correlation-ID** — Microsoft / 일부
- **traceparent / tracestate** — **W3C Trace Context** (모던 표준)

### W3C Trace Context

```http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

- OpenTelemetry 표준
- 분산 환경에서 trace 연결

---

## 6. X-API-Key

```http
X-API-Key: abcdef123456
```

- API 키 인증 (간단)
- Authorization 보다 약한 사용 (간단한 외부 API)
- HTTPS 필수

자세히 → [[../auth/api-keys]]

---

## 7. X-CSRF-Token

```http
X-CSRF-Token: <random-token>
```

- CSRF 방어
- 폼 / AJAX 요청에 동봉

자세히 → [[../security/security]]

---

## 8. X-Frame-Options (보안)

```http
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
X-Frame-Options: ALLOW-FROM https://example.com    (deprecated)
```

- Clickjacking 방어
- 모던: CSP `frame-ancestors` 가 대체

자세히 → [[../security/x-frame-options]]

---

## 9. X-Content-Type-Options

```http
X-Content-Type-Options: nosniff
```

- MIME sniffing 방어
- 거의 모든 사이트 활성

자세히 → [[../security/x-content-type-options]]

---

## 10. Rate Limit 헤더 (비표준)

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1715587200
Retry-After: 60          (표준)
```

### 표준화 작업
- RFC 9651 (Structured Field Values for HTTP)
- `RateLimit-*` (prefix 없이) 표준 작업 중

---

## 11. 회사 / 서비스 별 X- 헤더

### AWS
```http
X-Amz-Date: 20260513T120000Z
X-Amz-Security-Token: ...
X-Amz-Content-SHA256: ...
```

### GitHub
```http
X-GitHub-Event: push
X-GitHub-Delivery: abc-123
X-Hub-Signature-256: sha256=...
```

### Cloudflare
```http
CF-Connecting-IP: ...
CF-Ray: 7a8b9c0d1e2f3g4h-ICN
CF-IPCountry: KR
```

### Stripe
```http
Stripe-Signature: t=1715587200,v1=...
```

---

## 12. 함정

### 함정 1 — XFF 신뢰 X
사용자 임의 — Spoofing. Trusted proxy 만.

### 함정 2 — XFF 의 chain
한 개로 가정 X. 여러 proxy 거치면 콤마로 나열.

### 함정 3 — Forwarded vs X-Forwarded-*
실제는 X- 가 더 보편. 둘 다 처리.

### 함정 4 — X-Request-ID 의 추적
응용 / 로그 / 모니터링 모두 같은 ID 사용 — 통합 필요. W3C Trace Context 권장.

### 함정 5 — 옛 X-Frame-Options
deprecated — CSP frame-ancestors 사용.

### 함정 6 — X-API-Key 의 보안
HTTPS 필수. 평문 노출 X.

### 함정 7 — 응용이 모르는 X- 헤더
일부 라우터 / Proxy 가 폐기 — 응용까지 안 옴.

---

## 13. 학습 자료

- **RFC 6648** (X- prefix deprecation)
- **RFC 7239** (Forwarded)
- **W3C Trace Context** — https://www.w3.org/TR/trace-context/
- MDN HTTP Custom Headers

---

## 14. 관련

- [[headers]] — Headers hub
- [[request-headers]] / [[response-headers]]
- [[../auth/api-keys]] — X-API-Key
- [[../security/x-frame-options]] / [[../security/x-content-type-options]]
- [[../../load-balancing/load-balancing]] — LB 의 XFF 처리
