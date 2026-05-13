---
title: "Certificates & CA — X.509 / Let's Encrypt"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:55:00+09:00
tags:
  - network
  - tls
  - certificate
  - x509
  - ca
---

# Certificates & CA — X.509 / Let's Encrypt

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | X.509 / CA chain / Let's Encrypt |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**디지털 인증서** — 도메인 / 사람 / 응용의 신원을 공개키와 함께 CA 가 서명. X.509 표준.

---

## 2. X.509 구조

```
Certificate
├── Version (v1 / v2 / v3)
├── Serial Number
├── Signature Algorithm
├── Issuer (CA)
├── Validity (Not Before / Not After)
├── Subject (Owner)
├── Subject Public Key Info
│   ├── Algorithm
│   └── Public Key
├── Extensions (v3)
│   ├── Subject Alternative Name (SAN)
│   ├── Key Usage
│   ├── Extended Key Usage
│   ├── Basic Constraints
│   ├── Authority Key Identifier
│   ├── Subject Key Identifier
│   ├── CRL Distribution Points
│   ├── Authority Info Access (OCSP)
│   ├── Certificate Policies
│   └── Signed Certificate Timestamp (CT)
└── CA's Signature
```

---

## 3. Subject / SAN

### Subject (옛)
```
CN=example.com, O=Example Inc, L=San Francisco, ST=CA, C=US
```

### SAN — Subject Alternative Name (모던)
```
SAN: DNS:example.com,
     DNS:www.example.com,
     DNS:api.example.com,
     IP:1.2.3.4
```

→ 모던 브라우저 — **SAN 만 검증** (CN 무시).

---

## 4. Certificate Authority (CA) 종류

### Root CA
- 자체 서명 (self-signed)
- OS / 브라우저 의 trust store 에 내장
- 발급 X — Intermediate 발급만

### Intermediate CA
- Root CA 가 서명
- 실제 인증서 발급
- Root 보호 (offline / HSM)

### Subordinate CA / Issuing CA
- 다단계 chain
- 예: Root → Intermediate → Issuing → Server

---

## 5. Trust Store

```
브라우저 / OS:
- Mozilla CA Bundle (~150 CA)
- Microsoft Trusted Root
- Apple Trusted Root
- Google Chrome — Chrome Root Store
```

### 추가 가능
- 회사 internal CA 추가
- 개발 cert (mkcert, Caddy local)
- 단 — 사용자 / 디바이스 별 신뢰 추가

---

## 6. 검증 종류 (DV / OV / EV)

### DV — Domain Validation
- 도메인 소유 검증만 (HTTP-01, DNS-01)
- 가장 흔함, **무료** (Let's Encrypt)
- 발급 수 분

### OV — Organization Validation
- 회사 / 조직 정보 검증
- 발급 며칠
- 비용 ($100-500/년)

### EV — Extended Validation
- 강한 신원 검증
- 옛 브라우저 — green bar
- 모던 브라우저 — UI 차이 거의 X (사장 중)
- 비싸 ($300-1000+/년)

→ **EV 의 가치 ↓** — 모던은 DV (Let's Encrypt) 가 표준.

---

## 7. Let's Encrypt 혁명

### 출시 (2016)
- ISRG (Internet Security Research Group)
- EFF / Mozilla / Cisco / Akamai 후원
- **무료 + 자동 + 짧은 만료 (90 일)**

### ACME 프로토콜 (RFC 8555)
- 자동 인증서 발급 표준
- HTTP-01 / DNS-01 / TLS-ALPN-01 challenge

### Challenge

#### HTTP-01
```
1. ACME 클라 (certbot) → Let's Encrypt: 도메인 인증 요청
2. Let's Encrypt: 토큰 발급
3. 서버: http://example.com/.well-known/acme-challenge/<token> 에 토큰 응답
4. Let's Encrypt: HTTP 요청 → 토큰 검증
5. 발급
```

#### DNS-01
```
서버: DNS TXT 레코드에 토큰 추가
_acme-challenge.example.com TXT "token-..."

Let's Encrypt: DNS 조회 → 검증
→ wildcard cert (*.example.com) 가능
```

### 도구
- **certbot** — Let's Encrypt 의 공식
- **acme.sh** — 가벼운 shell
- **Caddy** — 자동 (built-in)
- **Traefik** — Docker / K8s
- **cert-manager** — Kubernetes

---

## 8. 다른 CA

### 무료
- **Let's Encrypt** — 가장 흔함
- **ZeroSSL** — 90 일 / 매뉴얼 OK
- **Buypass Go** — 노르웨이
- **Cloudflare Origin CA** — Cloudflare 만 신뢰

### 유료
- **DigiCert** — Enterprise / EV
- **GlobalSign**
- **Sectigo / Comodo**
- **GeoTrust**

---

## 9. Wildcard / Multi-domain

### Wildcard
```
*.example.com
→ a.example.com, b.example.com 매칭
→ x.y.example.com 매칭 X (단일 레벨)
```

### Multi-SAN (multi-domain)
```
SAN: DNS:example.com,
     DNS:www.example.com,
     DNS:api.example.com,
     DNS:other-domain.com
```

→ 한 cert 가 여러 도메인.

---

## 10. 인증서 형식 / 확장자

| 확장자 | 형식 |
| --- | --- |
| `.pem` | Base64 (BEGIN CERTIFICATE) — 가장 흔함 |
| `.crt` / `.cer` | PEM 또는 DER |
| `.der` | Binary DER |
| `.key` | 비밀키 (PEM) |
| `.csr` | Certificate Signing Request |
| `.p12` / `.pfx` | PKCS#12 (cert + key 함께, password 보호) |
| `.p7b` / `.p7c` | PKCS#7 chain 만 (key 없음) |

### 변환

```bash
# PEM → DER
openssl x509 -in cert.pem -outform DER -out cert.der

# DER → PEM
openssl x509 -in cert.der -inform DER -outform PEM -out cert.pem

# PFX → PEM + key
openssl pkcs12 -in cert.pfx -nocerts -out key.pem
openssl pkcs12 -in cert.pfx -nokeys -out cert.pem
```

---

## 11. CSR — Certificate Signing Request

```bash
# 키 + CSR 생성
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr \
  -subj "/CN=example.com"

# CSR 내용 확인
openssl req -in server.csr -text -noout
```

### 의미
- 공개키 + Subject + 자기 서명
- CA 에게 전달 → CA 가 검증 후 cert 발급

---

## 12. CT — Certificate Transparency

### RFC 6962
- 모든 발급 cert 가 공개 로그에 등록
- Google / Cloudflare / Sectigo 등 운영
- 잘못 발급된 cert 감지

### SCT (Signed Certificate Timestamp)
- Cert 의 extension 또는 TLS handshake 에 포함
- 클라가 검증 — Chrome 강제 (2018+)

### crt.sh
- https://crt.sh/ — CT 로그 검색
- 도메인의 모든 cert 조회
- 보안 감사 / 만료 모니터링

---

## 13. 검증 / 추출

```bash
# Cert 내용
openssl x509 -in cert.pem -text -noout

# Subject / SAN
openssl x509 -in cert.pem -noout -subject -issuer
openssl x509 -in cert.pem -noout -ext subjectAltName

# 만료
openssl x509 -in cert.pem -noout -enddate

# Fingerprint
openssl x509 -in cert.pem -noout -fingerprint -sha256

# 원격
openssl s_client -connect example.com:443 -servername example.com \
  </dev/null 2>/dev/null | openssl x509 -text
```

---

## 14. 함정

### 함정 1 — Cert chain 불완전
서버가 server cert 만 보내고 intermediate 누락 → 클라가 검증 X.
→ Let's Encrypt 등은 full chain 명시 (`fullchain.pem`).

### 함정 2 — CN 만 / SAN 누락
모던 — SAN 만 검증. CN 만 있으면 거부.

### 함정 3 — 만료
Let's Encrypt 90 일 — 자동 갱신 필수. Cron + certbot renew.

### 함정 4 — Private key 노출
HSM / 환경변수 / Vault. `.git` X.

### 함정 5 — Self-signed in production
브라우저 경고. Let's Encrypt 무료.

### 함정 6 — IP 인증서
가능하지만 흔하지 X. DNS 권장.

### 함정 7 — Wildcard 의 보안
한 cert 도난 → 모든 subdomain 영향. 분리 권장 (multi-SAN).

---

## 15. 학습 자료

- **RFC 5280** (X.509 PKI)
- **RFC 6962** (CT)
- **RFC 8555** (ACME)
- "Bulletproof SSL and TLS" Ch. 3
- Let's Encrypt docs

---

## 16. 관련

- [[tls-ssl]] — TLS hub
- [[certificate-validation]] — 검증 자세히
- [[../http/auth/mtls-cert-auth]] — 클라 인증서
- [[../../../security-theory/security-theory]] — PKI
