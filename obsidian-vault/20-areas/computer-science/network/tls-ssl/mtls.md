---
title: "mTLS — Mutual TLS (TLS 측면)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:05:00+09:00
tags:
  - network
  - tls
  - mtls
  - mutual-tls
---

# mTLS — Mutual TLS (TLS 측면)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLS handshake 의 client cert 부분 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

> 응용 / 사용 사례 깊이는 [[../http/auth/mtls-cert-auth|↗ HTTP auth mTLS]]. 이 노트는 **TLS handshake 의 mTLS 측면**.

---

## 1. 한 줄 정의

TLS handshake 의 옵션 단계 — **서버가 클라 cert 도 요구**. 응용 레벨 인증 없이 강한 신원.

---

## 2. TLS 1.2 의 mTLS Handshake

```
Client                                       Server
  |---- ClientHello ------------------------>    |
  |  <---- ServerHello                          |
  |  <---- Certificate                          |
  |  <---- ServerKeyExchange                    |
  |  <---- CertificateRequest -------------    |  ← mTLS 명시
  |        - cert_types: rsa_sign, ecdsa_sign   |
  |        - distinguished_names: 신뢰 CA      |
  |  <---- ServerHelloDone                      |
  |                                              |
  |---- Certificate ------------------------> |  ← 클라 cert
  |---- ClientKeyExchange ----------------> |
  |---- CertificateVerify ------------------> |  ← 클라 비밀키 의 서명
  |---- ChangeCipherSpec ----------------> |
  |---- Finished -----------------------------> |
  |  <---- ChangeCipherSpec                     |
  |  <---- Finished                              |
```

### CertificateRequest 의 정보
- 어떤 cert type 받을지 (rsa / ecdsa)
- 신뢰 CA 의 DN 목록 — 클라가 이 중 하나의 cert 보내야

### CertificateVerify
- 클라가 자기 비밀키로 handshake 의 hash 서명
- 서버가 클라 cert 의 공개키로 검증
- "이 cert 가 진짜 너 거구나" 증명

---

## 3. TLS 1.3 의 mTLS

```
Client                                       Server
  |---- ClientHello ------------------------>    |
  |  <---- ServerHello                          |
  |  <---- {EncryptedExtensions}                |
  |  <---- {CertificateRequest}                 |  ← mTLS 시
  |  <---- {Certificate}                          |
  |  <---- {CertificateVerify}                    |
  |  <---- {Finished}                             |
  |                                              |
  |---- {Certificate} ----------------------> |
  |---- {CertificateVerify} ---------------> |
  |---- {Finished} ----------------------------> |
```

→ 1 RTT — 모두 암호화.

### 차이
- TLS 1.2 는 클라 cert / verify 가 명시 메시지
- TLS 1.3 는 모두 암호화 (Application data 와 같이)

---

## 4. Client Cert 의 형식

```
Subject: CN=service-a.internal,
         O=Example Corp,
         OU=Engineering
Subject Alternative Name:
  - DNS:service-a.internal.company.com
  - URI:spiffe://company/service-a
  - email:user@example.com
Extended Key Usage:
  - TLS Web Client Authentication
```

### EKU — Extended Key Usage
- **id-kp-clientAuth** (1.3.6.1.5.5.7.3.2) — 클라 인증
- 서버 cert 와 분리

---

## 5. Server 의 검증 방식

### 옵션 1 — Verify 만
```nginx
ssl_verify_client on;
ssl_client_certificate /etc/ssl/ca.crt;
```

→ CA 가 서명한 모든 cert 통과.

### 옵션 2 — Verify + Application 로직
```nginx
ssl_verify_client on;
ssl_client_certificate /etc/ssl/ca.crt;

location / {
    proxy_pass http://backend;
    proxy_set_header X-Client-Cert-DN $ssl_client_s_dn;
    proxy_set_header X-Client-Cert-Verify $ssl_client_verify;
}
```

→ 응용이 DN / SAN / EKU 등 추가 검증.

### 옵션 3 — Optional
```nginx
ssl_verify_client optional;
```

→ 클라 cert 보내면 검증, 없으면 통과 (응용이 결정).

---

## 6. SPIFFE / SPIRE — Cloud Native Identity

### SPIFFE — Secure Production Identity Framework For Everyone
- Cloud-native cert 표준
- SPIFFE ID 형식: `spiffe://trust-domain/workload-id`
- 예: `spiffe://company.com/ns/prod/sa/payment-service`

### SPIRE — SPIFFE 의 구현
- Agent + Server
- Workload 자동 cert 발급
- Kubernetes / VM / Lambda

### 사용
- Istio / Linkerd / Consul Connect
- 자동 mTLS

---

## 7. cert-manager (Kubernetes)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: service-a
spec:
  secretName: service-a-tls
  issuerRef:
    name: internal-ca
  dnsNames:
    - service-a.example.com
  duration: 24h            # 짧은 만료
  renewBefore: 2h
```

→ 자동 발급 + 갱신.

---

## 8. Performance

### Handshake 비용
- 추가 1 RTT (TLS 1.2) — 클라 cert + verify
- TLS 1.3 — 같은 1 RTT 안

### Asymmetric ops
- 클라 + 서버 양쪽 ECDSA / RSA 서명 + 검증
- HW 가속 가능 (HSM)

### 세션 재사용 (Session Resumption)
- 처음 mTLS 후 PSK / Ticket — 재사용 시 cert 검증 X
- 큰 부하 감소

---

## 9. mTLS vs JWT vs API Key

자세히 → [[../http/auth/mtls-cert-auth]]

| 측면 | mTLS | JWT | API Key |
| --- | --- | --- | --- |
| 강도 | 매우 강 | 강 | 보통 |
| Service Mesh | ✅ | ✅ | ❌ |
| 사용자 | 어려움 | 좋음 | — |
| 운영 | 복잡 | 보통 | 단순 |

---

## 10. 함정

### 함정 1 — Cert 만료
짧은 만료 + 자동 갱신 (cert-manager / SPIRE).

### 함정 2 — Private key HSM
HSM / TPM. 디스크 평문 X.

### 함정 3 — Verify optional 의 함정
응용이 cert 없는 경우 처리 안 함 → 인증 우회.

### 함정 4 — CRL 의 한계
큰 환경 — Short-lived cert.

### 함정 5 — Cipher Suite 호환
ECDSA cert 면 ECDSA cipher suite 필요. RSA fallback 도.

### 함정 6 — 클라 cert 의 EKU
**TLS Web Client Authentication** 명시. 서버 cert 와 다름.

---

## 11. 학습 자료

- **RFC 5246** (TLS 1.2 mTLS), **RFC 8446** (TLS 1.3)
- "Mutual TLS" — Cloudflare 가이드
- SPIFFE / SPIRE 공식 문서
- Istio mTLS

---

## 12. 관련

- [[tls-ssl]] — TLS hub
- [[tls-handshake-1-2]] / [[tls-1-3]]
- [[certificates-ca]] — Cert 자체
- [[../http/auth/mtls-cert-auth]] — 응용 / 사용 사례 자세히
