---
title: "API Keys"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:55:00+09:00
tags:
  - network
  - http
  - auth
  - api-key
---

# API Keys

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 위치 / 관리 / 보안 |

**[[auth|↑ Auth]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응용에 부여된 **장기 비밀 문자열** — 매 요청에 송신해 인증. OAuth 보다 단순.
외부 API 의 흔한 방식 (Stripe / Twilio / OpenAI 등).

---

## 2. 전송 위치

### 2.1 헤더 (권장)

```http
Authorization: Bearer sk-1234...
X-API-Key: abc123...
X-Auth-Token: ...
```

### 2.2 Query parameter (권장 X)

```
GET /api/users?api_key=abc123
```

→ URL 노출 (로그 / Referer / 브라우저 history). 비밀 정보 X.

### 2.3 Body

```json
POST /api { "api_key": "abc123", ... }
```

→ POST 만. 비표준.

---

## 3. 회사별 패턴

### Stripe
```http
Authorization: Bearer sk_test_<YOUR-KEY>
```

### OpenAI
```http
Authorization: Bearer sk-...
```

### AWS
```http
Authorization: AWS4-HMAC-SHA256 Credential=...,
               SignedHeaders=...,
               Signature=...
```

→ AWS 는 **HMAC 서명** — 키 자체 X. 더 안전.

### GitHub
```http
Authorization: Bearer ghp_abcdef...
Authorization: token ghp_...        ← 옛
```

### Twilio
```http
Authorization: Basic <Base64(AccountSid:AuthToken)>
```

→ Basic Auth + Account / Token.

### Algolia
```http
X-Algolia-Application-Id: APP_ID
X-Algolia-API-Key: KEY
```

---

## 4. Key 의 종류

### Public API Key
- 클라이언트 (브라우저) 에서 사용 OK
- Search / Map / Analytics
- 도메인 제한 + Rate limit

### Private / Secret API Key
- 서버 측만
- 절대 클라이언트 노출 X
- 환경변수 / Secret manager

### Test / Production Keys
```
Stripe:
  sk_test_...     ← 테스트
  sk_live_...     ← 프로덕션
```

---

## 5. 키 관리

### 5.1 생성
- 강한 random (32+ byte)
- 접두사 (`sk_live_`) 로 식별

### 5.2 저장
- Hash 만 저장 (bcrypt / argon2) — 의미적 보호
- 또는 암호화 (KMS)
- 사용자가 보는 건 원본 (한 번만)

### 5.3 Rotation
- 정기 갱신 (1 년)
- 도난 의심 시 즉시
- Grace period (옛 + 새 동시)

### 5.4 Revocation
- 즉시 무효화
- 사용자 dashboard
- 보안 사고 대응

### 5.5 Audit
- 사용 로그
- 비정상 패턴 감지

---

## 6. Scope / Permissions

```
Read-only key:  read scope
Admin key:      all scopes
Webhook key:    receive scope
```

### 예 — GitHub
```
read:user, repo, write:org, ...
```

→ 최소 권한 원칙.

---

## 7. Rate Limit

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1715587200

또는 429:
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

→ 키 별 / 응용 별 / IP 별.

---

## 8. 보안 위험

### 8.1 GitHub 등에 commit
- 가장 흔한 사고 — 코드 저장소에 API key
- 자동 스캐너 (Bot / GitHub Secret Scan)
- 즉시 회전 + 도용 감사

### 8.2 클라이언트 노출
- JS / Mobile 에 secret key — XSS / decompile
- Public key 만 클라이언트

### 8.3 환경변수 누출
- `.env` 파일 git commit
- 로그에 출력
- 에러 메시지

### 8.4 HTTP 평문
- HTTPS 필수
- Public Wi-Fi 도청

---

## 9. 모범 사례

### 9.1 Server-side only (secret key)
```python
# 환경변수
import os
api_key = os.environ['STRIPE_SECRET_KEY']

# 또는 Secret Manager
from boto3 import client
sm = client('secretsmanager')
api_key = sm.get_secret_value(SecretId='stripe')['SecretString']
```

### 9.2 .gitignore
```
.env
.env.local
secrets/
```

### 9.3 Pre-commit hooks
- gitleaks / trufflehog
- 커밋 전 비밀 검사

### 9.4 Token rotation 자동
- AWS Secrets Manager
- HashiCorp Vault

### 9.5 Granular keys
- Read-only 와 Admin 분리
- 환경 (dev / staging / prod) 분리
- 사용자 / 응용별

---

## 10. API Key vs Bearer Token vs Session

| 측면 | API Key | Bearer (JWT) | Session Cookie |
| --- | --- | --- | --- |
| 수명 | 매우 김 (1년+) | 짧음 (분) | 중 (시간) |
| 회전 | 수동 / 가끔 | 자동 (Refresh) | 자동 |
| 사용자 | 응용 | 사용자 / 응용 | 사용자 |
| 무효화 | DB 검색 | 만료 / Blacklist | 즉시 |
| 사용 | 외부 API | Web/Mobile | Web |

---

## 11. JWT in API Key 패턴

일부 응용 — JWT 같은 형식의 API Key:
```
sk_live_eyJhbGc...   (Stripe 의 짧은 식별자 + JWT)
```

→ JWT 의 stateless 검증 + Key 의 단순 관리.

---

## 12. 함정

### 함정 1 — 코드 저장소 노출
- 즉시 회전 + 로그 감사
- `git rebase` 안 됨 — 이미 노출됨
- Secret Scan 활성

### 함정 2 — Public / Private 혼동
- Public key 가 secret 동작 시도
- Private key 가 클라이언트에

### 함정 3 — 환경별 같은 키
- dev / staging / prod 다른 키
- 사고 격리

### 함정 4 — 무한 수명
- Rotation 정책
- 사용 안 함 → 비활성

### 함정 5 — 로그에 출력
- masking (`sk_live_***...`)
- Sentry / Datadog 등 — 자동 sanitize

### 함정 6 — Body 에서 검증
- 헤더 권장 — Body parsing 전 인증

### 함정 7 — Constant-time 비교
- `==` 비교 — timing attack
- `hmac.compare_digest()` (Python) 등

---

## 13. 학습 자료

- "API Key Best Practices" — Google Cloud
- "Securing API Keys" — Mozilla
- AWS / Stripe API Key 가이드

---

## 14. 관련

- [[auth]] — Auth hub
- [[bearer-token]] — Bearer (JWT)
- [[../../../security-theory/security-theory]] — 비밀 관리
