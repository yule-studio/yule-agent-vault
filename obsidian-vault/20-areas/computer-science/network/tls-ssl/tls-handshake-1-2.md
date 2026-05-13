---
title: "TLS 1.2 Handshake (Full + Resumed)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:40:00+09:00
tags:
  - network
  - tls
  - tls-1-2
  - handshake
---

# TLS 1.2 Handshake (Full + Resumed)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 2-RTT handshake + 키 교환 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

TLS 1.2 의 **2-RTT** handshake — 4 단계 (ClientHello → ServerHello+Cert → KeyExchange → Finished).
RFC 5246 (2008).

---

## 2. Full Handshake (2 RTT)

```
Client                                       Server
  |                                              |
  |---- ClientHello ------------------------>    |  RTT 1
  |     - version: TLS 1.2                       |
  |     - cipher_suites: [...]                   |
  |     - extensions (SNI, ALPN, ...)            |
  |     - random (client)                        |
  |                                              |
  |  <---- ServerHello -----------------------|  RTT 1
  |        - version, cipher chosen               |
  |        - random (server)                      |
  |  <---- Certificate ---------------------     |
  |        - server cert + chain                  |
  |  <---- ServerKeyExchange ------------         |
  |        - DH params, signature (ECDHE)         |
  |  <---- (CertificateRequest)                   |  mTLS 시
  |  <---- ServerHelloDone --------------         |
  |                                              |
  |---- (Certificate) ----------------------->   |  RTT 2
  |     - client cert (mTLS)                      |
  |---- ClientKeyExchange -------------------->   |  RTT 2
  |     - DH public key                           |
  |---- ChangeCipherSpec ----------------------> |
  |---- Finished (encrypted) ---------------->   |
  |                                              |
  |  <---- ChangeCipherSpec ---------          |
  |  <---- Finished (encrypted) -------         |
  |                                              |
  |== 양방향 암호화 통신 시작 ============== |
```

---

## 3. 각 단계 자세히

### 3.1 ClientHello

```
TLS Version: 1.2
Random (32 byte)
Session ID (재방문)
Cipher Suites: [
    TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
    TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
    ...
]
Compression: [null]
Extensions: [
    server_name (SNI),
    supported_groups (ECC curves),
    signature_algorithms,
    application_layer_protocol_negotiation (ALPN),
    ...
]
```

### 3.2 ServerHello

```
TLS Version: 1.2
Random (32 byte)
Session ID
Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384  ← 선택
Extensions: [
    ALPN: "h2",
    ...
]
```

### 3.3 Certificate

```
X.509 cert chain:
  Server Cert (CN: example.com)
    ← signed by Intermediate CA
    ← signed by Root CA
```

→ 클라가 PKI 체인 검증.

자세히 → [[certificates-ca]]

### 3.4 ServerKeyExchange (ECDHE)

```
Curve: secp256r1 (P-256)
Server EC Public Key
Signature: RSA-PSS or ECDSA over (random_c | random_s | params)
```

→ Forward Secrecy 의 기반 — DH 의 비밀 키는 임시 (ephemeral).

### 3.5 ClientKeyExchange

```
Client EC Public Key
```

→ 양쪽 ECDHE 의 공유 비밀 (pre-master secret) 계산.

### 3.6 키 도출

```
pre-master secret (ECDHE)
       ↓
master secret = PRF(pre-master, "master secret", random_c | random_s)
       ↓
key block = PRF(master, "key expansion", random_s | random_c)
       ↓
client_write_key, server_write_key,
client_write_IV, server_write_IV
```

### 3.7 ChangeCipherSpec + Finished

- 이후 모든 메시지 암호화
- Finished = MAC over all handshake messages (정합성)

---

## 4. Cipher Suite 의 의미

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    │    │    │      │       │       │
    │    │    │      │       │       └─ HMAC 의 hash
    │    │    │      │       └─ AEAD mode
    │    │    │      └─ Symmetric cipher
    │    │    └─ "with"
    │    └─ Authentication (서버 cert)
    └─ Key exchange (ECDHE — Forward Secrecy)
```

자세히 → [[cipher-suites]]

---

## 5. 키 교환 종류

### ECDHE (Ephemeral Elliptic Curve DH) — 권장
- Forward Secrecy
- 매 세션 새 DH 키
- TLS 1.3 의 필수

### DHE (Ephemeral DH)
- Forward Secrecy
- 느림 (큰 prime)
- 사실상 ECDHE

### RSA (key exchange) — deprecated
- 서버의 RSA private key 로 pre-master 복호화
- Forward Secrecy X (key 도난 시 모든 옛 트래픽 복호화)
- **TLS 1.3 제거**

### PSK (Pre-Shared Key)
- 사전 공유 비밀
- IoT / 제한 환경

---

## 6. Renegotiation (TLS 1.2 — TLS 1.3 제거)

```
ESTABLISHED 후 새 handshake 시작:
- Cipher 변경
- Cert 변경 (예: 인증 추가)

→ Renegotiation attack (Marsh Ray 2009) — RFC 5746 패치
→ TLS 1.3 에서 완전 제거
```

---

## 7. Resumed Handshake (1 RTT)

### Session ID 방식

```
Client                                       Server
  |---- ClientHello + Session ID -------->   |  RTT 1
  |  <---- ServerHello + Session ID --       |
  |  <---- ChangeCipherSpec + Finished --   |
  |                                          |
  |---- ChangeCipherSpec + Finished ----->   |  RTT 1 끝
```

→ 1 RTT — 서버가 session ID 매칭 시.

### Session Ticket 방식 (RFC 5077)

```
서버가 클라에 암호화된 ticket 발급
다음 연결 시 클라가 ticket 보냄 → 서버가 복호화 → 즉시 resumed
```

자세히 → [[session-resumption]]

---

## 8. False Start (옛 최적화)

```
Client Finished 보낸 후 Server Finished 안 기다리고 데이터 송신
→ 1 RTT 절약
```

→ 보안 문제로 거의 사용 X. TLS 1.3 이 표준으로 1-RTT.

---

## 9. tcpdump / Wireshark 로 보기

```bash
sudo tcpdump -i en0 -nn -X 'tcp port 443'

# Wireshark display filter
tls.handshake.type == 1     (ClientHello)
tls.handshake.type == 2     (ServerHello)
tls.handshake.type == 11    (Certificate)
tls.handshake.type == 14    (ServerHelloDone)
tls.handshake.type == 16    (ClientKeyExchange)
tls.handshake.type == 20    (Finished)
```

### SSLKEYLOGFILE — 복호화

```bash
export SSLKEYLOGFILE=~/ssl-keys.log
curl https://example.com/

# Wireshark — Preferences → Protocols → TLS → (Pre)-Master-Secret log
```

---

## 10. openssl s_client

```bash
openssl s_client -connect example.com:443 \
  -servername example.com \
  -tls1_2 \
  -showcerts

# 출력:
# - CONNECTED(00000003)
# - depth=2, 1, 0 — cert chain
# - SSL handshake details
# - Cipher chosen
# - Session ID
```

---

## 11. 함정

### 함정 1 — Forward Secrecy 없는 cipher
TLS_RSA_WITH_... — Forward Secrecy X. ECDHE 사용.

### 함정 2 — SNI 미설정
한 IP 의 여러 도메인 — 잘못된 cert 응답.

### 함정 3 — Cert chain 누락
서버가 intermediate cert 안 보냄 → 클라가 chain 검증 X. Let's Encrypt 의 chain 명시.

### 함정 4 — Renegotiation 활성
TLS 1.3 권장. 1.2 의 secure renegotiation (RFC 5746) 만.

### 함정 5 — Session Ticket key rotation 없음
ticket key 노출 → 모든 옛 트래픽 복호화. 정기 회전 (24 시간).

---

## 12. 학습 자료

- **RFC 5246** (TLS 1.2)
- **Bulletproof SSL and TLS** Ch. 2
- "The First Few Milliseconds of an HTTPS Connection" — Jeff Moser

---

## 13. 관련

- [[tls-ssl]] — TLS hub
- [[tls-1-3]] — 1.3 비교
- [[cipher-suites]] — Cipher Suite
- [[certificates-ca]] — Certificate
- [[session-resumption]]
