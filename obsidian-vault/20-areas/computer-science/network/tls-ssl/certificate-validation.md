---
title: "Certificate Validation — 체인 / OCSP / CT / Pinning"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:00:00+09:00
tags:
  - network
  - tls
  - certificate
  - ocsp
  - revocation
---

# Certificate Validation — 체인 / OCSP / CT / Pinning

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 검증 흐름 + 폐기 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 검증 흐름 — 6 단계

```
1. 서버 cert 받음
2. 체인 검증 (Root 까지)
3. 서명 검증 (각 단계 CA 의 서명)
4. 유효 기간 (Not Before / Not After)
5. 도메인 매칭 (SAN)
6. 폐기 여부 (CRL / OCSP)
+ Certificate Transparency (CT) — 모던 추가
```

→ 하나라도 실패 = 연결 거부.

---

## 2. 체인 검증

```
Server Cert (example.com)
   ↓ signed by
Intermediate CA (R3)
   ↓ signed by
Root CA (ISRG Root X1)
   ↓ self-signed
(클라의 Trust Store 에 있음)
```

### 검증
- 각 단계 CA 의 공개키로 서명 검증
- Root 가 Trust Store 에 있어야 신뢰

### Cross-signing (옛 호환)
- 새 Root CA 가 옛 Root CA 에게도 서명받음
- 옛 Trust Store 가진 클라 (옛 Android 등) 호환
- Let's Encrypt ISRG X1 ← 옛 IdenTrust DST X3 cross-sign (2024 만료)

---

## 3. 도메인 매칭 (SAN)

### 규칙
```
Cert SAN: DNS:example.com, DNS:*.example.com

요청 도메인 매칭:
  example.com           → DNS:example.com ✓
  www.example.com       → DNS:*.example.com ✓
  api.example.com       → DNS:*.example.com ✓
  www.api.example.com   → ✗ (wildcard 는 한 레벨)
  other.com             → ✗
```

### IDN (국제화 도메인)
- Punycode 변환 — `한글.com` → `xn--bj0bj06e.com`
- Cert 도 Punycode 형식

---

## 4. 폐기 — CRL / OCSP

### Cert 폐기 이유
- Private key 도난
- 회사 파산 / 도메인 양도
- 잘못 발급 (CA 실수)
- 알고리즘 약화

### 4.1 CRL (Certificate Revocation List)

```
CA → 큰 폐기 cert 목록 (PEM 또는 DER)
URL 은 cert 의 CRL Distribution Points extension 에
```

#### 단점
- 큰 파일 (수십 MB) — 매 연결 다운로드 비실용
- 옛 — 잘 사용 X

### 4.2 OCSP (Online Certificate Status Protocol)

```
클라 → OCSP responder: "이 cert 유효해?"
OCSP responder → 클라: good / revoked / unknown
```

#### 단점
- 매 연결 추가 RTT
- 프라이버시 — CA 가 사용자 행동 추적
- OCSP responder 가 down 시 hard-fail / soft-fail 트레이드오프

### 4.3 OCSP Stapling (RFC 6066)

```
서버가 미리 OCSP 응답 받아 → TLS handshake 에 첨부
→ 클라가 별도 OCSP 요청 X
```

#### 장점
- 빠름 (RTT X)
- 프라이버시 (CA 가 클라 IP 모름)
- 모던 표준

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/ssl/chain.pem;
resolver 8.8.8.8 1.1.1.1;
```

### 4.4 Must-Staple (RFC 7633)

Cert 에 extension — "이 cert 는 항상 stapled OCSP 첨부 필수":
```
TLS Feature: status_request
```

→ Stapling 누락 시 클라가 거부. 옛 — 잘 채택 X.

---

## 5. 모던 폐기 — Short-lived Cert

```
짧은 만료 (수 시간 - 7 일) cert
→ 폐기 검증 불필요 (만료가 곧 폐기)
```

### 예
- **mTLS** — 디바이스 cert
- **SPIFFE** — Kubernetes
- **AWS** internal

→ ACME / cert-manager 의 자동 갱신.

---

## 6. CT — Certificate Transparency

자세히 → [[certificates-ca#12 CT]]

### 검증 (Chrome 2018+)
- Cert 에 **SCT** (Signed Certificate Timestamp) 필요
- 2 개 이상 CT log 의 SCT
- 없으면 NET::ERR_CERTIFICATE_TRANSPARENCY_REQUIRED

### 발급 시
- CA 가 cert 를 CT log 에 제출 → SCT 받음
- Cert 에 embed 또는 TLS handshake / OCSP 에서 첨부

---

## 7. Certificate Pinning

### HPKP (옛, deprecated)
```http
Public-Key-Pins: pin-sha256="..."; max-age=...
```

→ 위험 (잘못된 pin = 영구 차단) — 사장.

### 모바일 / Native
- Android Network Security Config
- iOS App Transport Security + TrustKit
- 앱이 expected pin / cert hash 보유 — 다른 cert 거부

### 사용
- 금융 앱 / 의료
- High-value targets
- MITM 방어 (corporate proxy 까지)

---

## 8. 검증 실패 — 브라우저 / OS 응답

### Chrome
- `NET::ERR_CERT_AUTHORITY_INVALID`
- `NET::ERR_CERT_DATE_INVALID`
- `NET::ERR_CERT_COMMON_NAME_INVALID`
- `NET::ERR_CERT_REVOKED`
- `NET::ERR_CERTIFICATE_TRANSPARENCY_REQUIRED`

### 사용자 UX
- 빨간 화면 + 경고
- "Advanced" → "Proceed anyway" (옵션)
- HSTS preload 시 "Proceed" 옵션 X

---

## 9. 디버깅

### openssl s_client
```bash
openssl s_client -connect example.com:443 \
  -servername example.com \
  -showcerts \
  -status      # OCSP

# 출력:
# - Verify return code: 0 (ok)   ← 성공
# - Verify return code: 21 (...) ← 실패 이유
```

### curl
```bash
curl -v https://example.com/
# - "SSL certificate problem: certificate has expired"
# - "SSL certificate problem: unable to get local issuer certificate"

# 무시 (개발)
curl -k https://...
```

### Python requests
```python
import requests
r = requests.get('https://example.com/', verify=True)
# verify=False — 무시 (개발만)
# verify='/path/to/ca-bundle.pem' — 커스텀 CA
```

---

## 10. 흔한 오류 / 해결

### 10.1 unable to get local issuer certificate
- 서버가 intermediate cert 안 보냄
- 해결: 서버 설정에 fullchain (서버 + intermediate) 명시

### 10.2 certificate has expired
- 만료 — 갱신
- Let's Encrypt: `certbot renew`

### 10.3 self signed certificate in certificate chain
- 자체 서명 cert 사용
- Production — Let's Encrypt
- 개발 — `mkcert` 또는 신뢰 추가

### 10.4 hostname mismatch
- SAN 에 도메인 누락
- Cert 재발급

### 10.5 unable to verify the first certificate
- intermediate 누락 (Apple / Node.js 의 default)
- `fullchain.pem` 사용

---

## 11. 함정

### 함정 1 — verify=False 사용
개발에서만. Production 절대 X — MITM 위험.

### 함정 2 — Cert 만료 모니터링
자동 갱신 + 알람 (만료 30 일 전).

### 함정 3 — OCSP 의 soft-fail
일부 환경 — OCSP responder down 시 무시 (보안 ↓).

### 함정 4 — Cross-signing 만료
Let's Encrypt 의 DST X3 cross-sign 2024 만료 — 옛 Android < 7.1 영향.

### 함정 5 — CT 미포함
새 cert (Chrome 2018+) — CT 없으면 차단.

### 함정 6 — Wildcard 의 너무 광범위
`*.example.com` 도난 시 모든 subdomain. 분리.

---

## 12. 학습 자료

- **RFC 5280** (X.509 / PKI)
- **RFC 6960** (OCSP), **RFC 6066** (OCSP Stapling)
- **RFC 6962** (CT)
- "Bulletproof SSL and TLS" Ch. 4

---

## 13. 관련

- [[tls-ssl]] — TLS hub
- [[certificates-ca]] — Cert 자체
- [[tls-handshake-1-2]] / [[tls-1-3]] — handshake 에서 검증
- [[../http/auth/mtls-cert-auth]] — 클라 cert
