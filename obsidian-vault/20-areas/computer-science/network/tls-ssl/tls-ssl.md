---
title: "TLS / SSL — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:30:00+09:00
tags:
  - network
  - tls
  - ssl
  - https
---

# TLS / SSL — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLS hub + 8 세부 |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-6-presentation/layer-6-presentation|↑↑ L6 Presentation]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L6/L5 (학문) / L4-L7 사이 (실제) |
| **포트** | 443 (HTTPS), 853 (DoT), 5223 (XMPP) 등 |
| **버전** | SSL 2.0/3.0 (사장), TLS 1.0/1.1 (사장), TLS 1.2/1.3 (현역) |
| **표준** | RFC 5246 (1.2), RFC 8446 (1.3) |
| **출처** | Netscape 1994 SSL → IETF 1999 TLS |

---

## 1. 한 줄 정의

**T**ransport **L**ayer **S**ecurity — TCP / UDP 위에서 **기밀성 + 무결성 + 인증** 을 제공.
HTTPS / SMTPS / IMAPS / LDAPS 등 모든 "S" 의 정체.

---

## 2. 보장하는 3 가지

| 속성 | 의미 | 메커니즘 |
| --- | --- | --- |
| **Confidentiality (기밀성)** | 도청 방지 | AEAD 암호 (AES-GCM, ChaCha20-Poly1305) |
| **Integrity (무결성)** | 변조 감지 | MAC / AEAD tag |
| **Authentication (인증)** | 서버 / (옵션) 클라 진위 | X.509 인증서 + CA chain |

---

## 3. 진화

| 버전 | 출시 | 상태 |
| --- | --- | --- |
| **SSL 1.0** | 1994 | 비공개 (보안 결함) |
| **SSL 2.0** | 1995 | RFC 6176 — 금지 |
| **SSL 3.0** | 1996 | POODLE (2014) — 금지 |
| **TLS 1.0** | 1999 | RFC 8996 (2021) — 폐기 |
| **TLS 1.1** | 2006 | RFC 8996 — 폐기 |
| **TLS 1.2** | 2008 | RFC 5246 — 현역 |
| **TLS 1.3** | 2018 | RFC 8446 — 모던 표준 |

자세히 → [[tls-history-ssl-deprecated]]

---

## 4. TLS 1.2 vs 1.3 한눈

| 측면 | TLS 1.2 | TLS 1.3 |
| --- | --- | --- |
| Handshake RTT | 2 | **1** (0-RTT 가능) |
| Cipher Suite 수 | 300+ | **5 개만** (모두 AEAD) |
| Forward Secrecy | 옵션 (RSA key exchange 면 X) | **강제** |
| 비대칭 키 교환 | RSA, DHE, ECDHE | **(EC)DHE 만** |
| 압축 | 지원 (CRIME 공격) | **제거** |
| Renegotiation | 지원 | **제거** |
| Static RSA | 지원 | **제거** |
| MD5 / SHA-1 | 지원 | **제거** |

자세히 → [[tls-handshake-1-2]], [[tls-1-3]]

---

## 5. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[tls-history-ssl-deprecated]] | SSL 의 역사 + 모든 deprecated 버전 (POODLE / BEAST / FREAK / Logjam) |
| [[tls-handshake-1-2]] | TLS 1.2 의 4-step / 2-RTT handshake |
| [[tls-1-3]] | TLS 1.3 의 1-RTT + 0-RTT |
| [[cipher-suites]] | Cipher Suite 형식 / AEAD / FS / 권장 |
| [[certificates-ca]] | X.509 / Subject / SAN / CA / Let's Encrypt |
| [[certificate-validation]] | 체인 검증 / OCSP / CT / Pinning |
| [[mtls]] | Mutual TLS / 클라 인증서 |
| [[session-resumption]] | Session ID / Session Ticket / PSK (1.3) |

---

## 6. HTTPS = HTTP + TLS

```
1. TCP handshake (1 RTT)
2. TLS handshake (1 RTT TLS 1.3, 2 RTT TLS 1.2)
3. HTTP 요청 / 응답
```

자세히 → [[../http/versions/version-comparison]]

---

## 7. ALPN — Application-Layer Protocol Negotiation

TLS handshake 안 응용 프로토콜 협상:

```
ClientHello + ALPN: ["h3", "h2", "http/1.1"]
ServerHello + ALPN: "h2"
```

→ 한 포트 (443) 에서 HTTP/1.1, HTTP/2, HTTP/3 결정.

---

## 8. SNI — Server Name Indication

```
ClientHello + server_name: "api.example.com"
```

### 의미
- 한 IP 의 여러 도메인 (Virtual Hosting) — TLS 도 도메인 별 인증서
- 서버가 어느 인증서 보낼지 결정

### ESNI / ECH (Encrypted ClientHello)
- SNI 자체가 평문 → 도메인 노출 (검열)
- **ECH** (Encrypted Client Hello) — Cloudflare / Firefox / Chrome
- 2024+ 점진 도입

---

## 9. 주요 공격 / 사고

| 공격 | 영향 | 방어 |
| --- | --- | --- |
| **POODLE** (2014) | SSL 3.0 padding | TLS 1.0+ |
| **BEAST** (2011) | TLS 1.0 CBC | TLS 1.1+ |
| **FREAK** (2015) | 약한 export cipher | RSA 1024+ |
| **Logjam** (2015) | DH 1024 | DH 2048+ |
| **Heartbleed** (2014) | OpenSSL buffer over-read | OpenSSL 패치 |
| **CRIME** (2012) | TLS compression | 압축 제거 |
| **BREACH** (2013) | HTTP compression + 비밀 | compress 끄기 / 분리 |
| **DROWN** (2016) | SSLv2 cross-protocol | SSLv2 제거 |
| **ROBOT** (2017) | RSA key exchange | RSA 제거 (TLS 1.3) |
| **Bleichenbacher** | PKCS#1 v1.5 padding | OAEP / TLS 1.3 |

자세히 → [[tls-history-ssl-deprecated]]

---

## 10. 인증서 발급 — Let's Encrypt 혁명

```
2014 EFF / Mozilla / Cisco → Let's Encrypt
2016 베타 → 무료 자동 인증서
2024 — 인터넷 HTTPS 의 절반+
```

### ACME 프로토콜 (RFC 8555)
- 자동 발급 + 갱신 (90 일)
- HTTP-01 / DNS-01 challenge
- certbot / acme.sh / Caddy 자동

자세히 → [[certificates-ca]]

---

## 11. 도구

```bash
# 인증서 / TLS 검증
openssl s_client -connect example.com:443 -servername example.com
openssl x509 -in cert.pem -text -noout

# SSL Labs (외부)
https://www.ssllabs.com/ssltest/

# testssl.sh
./testssl.sh https://example.com

# nmap
nmap --script ssl-enum-ciphers -p 443 example.com

# curl
curl -v --tlsv1.3 https://example.com/
```

---

## 12. 면접 / 토픽

1. **TLS 1.2 vs 1.3** handshake 차이.
2. **TLS 의 3 가지 보장** (CIA).
3. **SNI / ESNI / ECH**.
4. **인증서 검증 흐름**.
5. **Forward Secrecy**.
6. **0-RTT 의 위험** (Replay attack).
7. **mTLS** 사용 사례.

---

## 13. 학습 자료

- **RFC 5246** (TLS 1.2) / **RFC 8446** (TLS 1.3)
- **Bulletproof SSL and TLS** (Ivan Ristić)
- **High Performance Browser Networking** Ch. 4
- Cloudflare blog TLS 시리즈
- mozilla.org/wiki/Security/Server_Side_TLS

---

## 14. 관련

- [[../http/http]] — HTTPS = HTTP + TLS
- [[../osi-7-layer/layer-6-presentation/layer-6-presentation]] — L6
- [[../quic/quic]] — QUIC 내장 TLS 1.3
- [[../../security-theory/security-theory]] — 암호학
