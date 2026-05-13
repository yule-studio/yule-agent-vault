---
title: "mTLS — Mutual TLS (Client Certificate)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:00:00+09:00
tags:
  - network
  - http
  - auth
  - mtls
  - tls
---

# mTLS — Mutual TLS (Client Certificate)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | mTLS / Zero Trust / Service Mesh |

**[[auth|↑ Auth]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

서버 + **클라이언트도 TLS 인증서** 로 인증. 양방향 TLS. 매우 강력한 인증 — 금융 /
정부 / Service Mesh.

---

## 2. 일반 TLS vs mTLS

### 일반 TLS
```
1. Client → Server: ClientHello
2. Server → Client: Certificate (서버 인증)
3. Client: 서버 cert 검증 (CA 체인)
4. ... handshake ...
5. 클라이언트 인증은 응용 레벨 (비밀번호 / 토큰)
```

### mTLS
```
1. Client → Server: ClientHello
2. Server → Client: Certificate + CertificateRequest
3. Client → Server: Certificate (클라 인증)
4. 양쪽 검증
5. ... handshake ...
6. 응용 레벨 인증 불필요 (TLS 가 신원 확인)
```

---

## 3. 동기 — 강력한 신원 확인

### 비밀번호 / 토큰의 한계
- 도난 위험
- 사람이 기억
- Phishing 가능

### 인증서
- 비밀키 = 디바이스 / 응용 의 hardware-bound
- 도난 어려움 (Secure Enclave / TPM)
- Phishing 면역

---

## 4. 사용 사례

### 4.1 금융 / 정부
- 은행 API
- 정부 시스템

### 4.2 Service Mesh
- Istio / Linkerd / Consul
- 모든 service-to-service mTLS
- Zero Trust

### 4.3 Mobile / IoT
- 디바이스 인증
- 출고 시 인증서 발급

### 4.4 VPN / Zero Trust
- BeyondCorp (Google) — 사용자 + 디바이스 인증서
- Cloudflare Access

### 4.5 API Gateway
- 외부 파트너 인증
- B2B integration

---

## 5. 인증서 발급 / 관리

### 5.1 PKI (Public Key Infrastructure)
- CA (Certificate Authority) — 인증서 발급
- 클라 비밀키 + 인증서 (CA 서명)
- CRL / OCSP — 폐기

### 5.2 Self-signed CA (내부)
- 회사 자체 CA 운영
- 모든 service / device 의 인증서 발급
- HashiCorp Vault, Smallstep, AWS Private CA

### 5.3 SPIFFE / SPIRE
- Cloud-native identity
- Service 자동 인증서 발급
- Kubernetes 통합

---

## 6. 흐름 (자세히)

```
1. CA → Service A: certificate + private key
2. CA → Service B: certificate + private key
3. CA 의 root certificate → 모든 service (truststore)

4. Service A → Service B: HTTPS request
   - ClientHello + Service A 의 cert
   - Service B: cert 검증 (CA 의 root 로)
   - Service B → Service A: cert
   - Service A: cert 검증
5. Handshake 완료
6. HTTP 통신
```

---

## 7. 응용 레벨 정보 (Cert 에서 추출)

### Subject / SAN
```
CN=service-a.internal.company.com
SAN: DNS:service-a, IP:10.0.0.1, URI:spiffe://company/service-a
```

→ 응용이 cert subject 로 신원 확인.

### Custom Extensions
- 역할 / 권한
- 만료
- 메타데이터

---

## 8. 서버 구현

### Nginx
```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/server.crt;
    ssl_certificate_key /etc/ssl/server.key;
    
    # mTLS
    ssl_client_certificate /etc/ssl/ca.crt;
    ssl_verify_client on;        # 또는 optional
    
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Client-Cert-Subject $ssl_client_s_dn;
        proxy_set_header X-Client-Cert-Verify $ssl_client_verify;
    }
}
```

### Apache
```apache
SSLCACertificateFile /etc/ssl/ca.crt
SSLVerifyClient require
SSLVerifyDepth 2
```

### Envoy / Istio
```yaml
peerAuthentication:
  mtls:
    mode: STRICT
```

→ 모든 인바운드 mTLS 강제.

---

## 9. 클라이언트 구현

### curl
```bash
curl --cert client.crt --key client.key --cacert ca.crt \
     https://api.example.com/
```

### Python requests
```python
import requests
r = requests.get(
    'https://api.example.com/',
    cert=('client.crt', 'client.key'),
    verify='ca.crt'
)
```

### Node.js
```javascript
const fs = require('fs');
const https = require('https');
const opts = {
    cert: fs.readFileSync('client.crt'),
    key: fs.readFileSync('client.key'),
    ca: fs.readFileSync('ca.crt'),
};
https.request('https://api.example.com/', opts);
```

### Java
```java
SSLContext.init(...) with KeyManagerFactory + TrustManagerFactory
```

---

## 10. Browser 의 mTLS

```
브라우저: 클라 cert prompt
사용자: 선택 (보유한 cert 중)
→ TLS handshake 에 cert 전송
```

### 단점
- 사용자 경험 ↓ (cert 설치 / 선택)
- 표준 UI 없음 — 옛 디자인
- 모바일 / 옛 브라우저 호환

→ 일반 웹 — 비밀번호 / OAuth. mTLS 는 enterprise / kiosk 일부.

---

## 11. mTLS vs 다른 방식

| 측면 | mTLS | OAuth / Bearer | Session Cookie |
| --- | --- | --- | --- |
| 강도 | 매우 강 | 강 | 보통 |
| 도난 위험 | 매우 낮음 (HW bound) | 보통 (토큰 도난) | 보통 |
| 사용자 경험 | 어려움 | 좋음 (SSO) | 좋음 |
| Mobile | 어려움 | 좋음 | 보통 |
| Service-to-Service | ✅ 표준 | ✅ Client Credentials | ❌ |
| 운영 | 복잡 (PKI) | 보통 | 단순 |

---

## 12. Service Mesh 의 mTLS

### Istio
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
spec:
  mtls:
    mode: STRICT
```

### Linkerd
- 모든 pod 의 sidecar (linkerd-proxy)
- 자동 cert 발급 + 회전
- Zero config

### Consul Connect
- HashiCorp
- 자동 mTLS

### 장점
- 응용 코드 X (sidecar 가 처리)
- 자동 cert 회전
- 통일된 보안

---

## 13. Certificate Pinning

```
클라이언트가 특정 cert / 공개키 hash 만 신뢰
→ CA 가 해킹돼도 안전
```

### HPKP (deprecated)
- HTTP Public Key Pinning
- 옛 — 브라우저 제거

### Mobile
- Android Network Security Config
- iOS TrustKit
- 모바일 앱 표준

---

## 14. 함정

### 함정 1 — Cert 만료
정기 갱신 — 자동화 필수. cert-manager (Kubernetes), Let's Encrypt.

### 함정 2 — Private key 노출
HSM (Hardware Security Module) 또는 TPM. 디스크 평문 X.

### 함정 3 — CA 의 해킹
모든 클라 영향. Private CA + 별도 보안.

### 함정 4 — CRL / OCSP 부담
- 큰 환경에서 폐기 검증 부하
- Short-lived cert (수 시간) 권장 — 폐기 불필요

### 함정 5 — Browser UX
사용자 cert 선택 — 혼란. 일반 웹 X.

### 함정 6 — Cert 의 cn vs SAN
모던 — SAN 만 검증 (CN 무시).

### 함정 7 — Self-signed CA 와 신뢰
모든 client 의 truststore 에 root CA 설치 필요. 큰 환경의 부담.

---

## 15. 학습 자료

- **RFC 8446** (TLS 1.3)
- "mTLS for Microservices" — Istio docs
- "SPIFFE — Identity for Cloud-Native"
- Google BeyondCorp 백서

---

## 16. 관련

- [[auth]] — Auth hub
- [[bearer-token]] — 다른 인증
- [[../../tls-ssl/tls-ssl]] — TLS 기본
- [[../../../security-theory/security-theory]] — Zero Trust / PKI
