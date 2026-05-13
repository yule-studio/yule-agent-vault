---
title: "TLS 1.3 — RFC 8446"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:45:00+09:00
tags:
  - network
  - tls
  - tls-1-3
  - handshake
---

# TLS 1.3 — RFC 8446

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 1-RTT + 0-RTT + Forward Secrecy 강제 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

TLS 의 모던 표준 — **1-RTT handshake** (재방문 0-RTT), **Forward Secrecy 강제**,
약한 cipher 모두 제거. 2018 RFC 8446.

---

## 2. TLS 1.2 vs 1.3 — 큰 변화

| 측면 | 1.2 | 1.3 |
| --- | --- | --- |
| Handshake | 2 RTT | **1 RTT** (재방문 **0 RTT**) |
| Cipher Suite | 300+ | **5** (모두 AEAD) |
| Key Exchange | RSA, DHE, ECDHE | **(EC)DHE 만** |
| Forward Secrecy | 옵션 | **강제** |
| MAC | HMAC | AEAD 내장 |
| Renegotiation | 지원 | **제거** |
| Compression | 지원 | **제거** (CRIME) |
| MD5/SHA-1 | 지원 | **제거** |
| RC4 / 3DES / DSA / Static RSA | 지원 | **모두 제거** |

---

## 3. 1-RTT Handshake

```
Client                                       Server
  |                                              |
  |---- ClientHello ------------------------>    |  RTT 1
  |     - cipher_suites (5 개)                   |
  |     - key_share (ECDHE public)               |
  |     - signature_algorithms                   |
  |     - supported_versions: 1.3                |
  |     - random                                 |
  |                                              |
  |  <---- ServerHello ----------------------    |  RTT 1
  |        - cipher chosen                        |
  |        - key_share (서버 ECDHE)              |
  |  <---- {EncryptedExtensions}                |
  |  <---- {Certificate}                          |  ← 암호화됨!
  |  <---- {CertificateVerify}                    |
  |  <---- {Finished}                             |
  |                                              |
  |---- {Finished} -------------------------> |
  |---- {Application Data} ----------------->    |  RTT 1 끝
  |  <---- {Application Data} ---------          |
```

→ 1 RTT 만에 양방향 통신.

### 차이 — 모든 cert / extension 암호화
- TLS 1.2: ServerHello 후 Certificate 평문
- TLS 1.3: Certificate 도 암호화 (개인 정보 보호)

---

## 4. Key Schedule

```
ECDHE 공유 비밀
    ↓ HKDF
Handshake Secret
    ↓ HKDF
client_handshake_traffic_secret
server_handshake_traffic_secret
    ↓ (handshake 완료 후) HKDF
Application Traffic Secret
    ↓
client_application_traffic_secret
server_application_traffic_secret
```

→ **HKDF** (HMAC-based Key Derivation, RFC 5869) 의 체인.

---

## 5. 5 가지 Cipher Suite (TLS 1.3)

```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

### 의미
- 모두 AEAD (Authenticated Encryption with Associated Data)
- Key Exchange / Signature 가 cipher suite 에서 분리
- 단순 + 안전

자세히 → [[cipher-suites]]

---

## 6. 0-RTT (Zero Round-Trip Time)

### 동작

```
첫 연결:
  Client ↔ Server: 1-RTT handshake
  Server → Client: NewSessionTicket
  Client: PSK (Pre-Shared Key) 저장

재방문:
  Client → Server: ClientHello + PSK + early_data (응용 데이터)
                                              ↑↑ 즉시!
  Server → Client: 응답 + Finished
```

→ 응용 데이터를 **첫 패킷에** 보냄. RTT 0.

### Early Data (0-RTT)

```
ClientHello:
  - psk_identity (Session Ticket)
  - early_data extension
  - application data (encrypted with PSK)
```

### Replay 위험

```
공격자가 0-RTT 패킷 캡처 → 다른 서버 / 시간에 재전송
→ 서버가 또 처리
```

### 방어
- **idempotent** 요청만 (GET 안전, POST 위험)
- 응용 / TLS 라이브러리가 anti-replay buffer
- Time-based — 짧은 시간만 0-RTT 수용

### 사용
- Cloudflare / Fastly 등 CDN
- TLS 1.3 의 큰 장점

---

## 7. PSK (Pre-Shared Key)

### 2 가지 PSK
- **External PSK** — 사전 설정 (IoT, 임베디드)
- **Resumption PSK** — 옛 세션의 ticket

### Resumption (TLS 1.2 의 Session ID/Ticket 대체)
- 모든 resumption 은 PSK 사용
- 더 단순 + 안전

---

## 8. Forward Secrecy 강제

### TLS 1.2
```
TLS_RSA_WITH_...    ← RSA key exchange — FS X
TLS_ECDHE_...       ← FS O
```

→ 옵션. 잘못 설정 시 FS X.

### TLS 1.3
```
모든 cipher 가 (EC)DHE 의무
```

→ 서버 private key 도난 시도 — 옛 세션 복호화 X.

---

## 9. PostQuantum TLS (실험)

### Hybrid Key Exchange
- ECDHE + 양자 내성 (Kyber 등)
- Cloudflare 2023+ 실험
- 양자 컴퓨터의 미래 위협 대응

### X25519Kyber768Draft00
- Chrome / Cloudflare 의 hybrid
- 모던 / 양자 모두 안전

---

## 10. Middlebox 호환성

```
TLS 1.3 packet 일부가 1.2 처럼 보임
- record version: TLS 1.2 (호환)
- ChangeCipherSpec dummy (옛 미들박스 호환)
- 진짜 1.3 은 supported_versions extension 으로
```

→ 옛 방화벽 / IDS 가 TLS 1.3 차단 회피.

---

## 11. SNI 의 평문 문제 — ECH

### 문제
- TLS 1.3 도 SNI (server_name) 가 평문
- 검열 / 추적 가능

### ECH (Encrypted Client Hello)
- ClientHello 자체를 암호화
- 공개키 (DNS HTTPS record) 로
- Cloudflare 2023, Firefox / Chrome 점진

---

## 12. 도입 현황 (2024)

| 서비스 | TLS 1.3 |
| --- | --- |
| Google / YouTube | ✅ default |
| Cloudflare 사이트 | ✅ default |
| Facebook / Meta | ✅ |
| Amazon / Microsoft | ✅ |
| Apple | ✅ |
| 일반 사이트 | 60-70% (점진) |

브라우저:
- Chrome / Firefox / Safari / Edge — 활성 (2018-2020)

---

## 13. openssl s_client (TLS 1.3)

```bash
openssl s_client -connect example.com:443 -tls1_3

# 출력 (관련):
# - SSL handshake has read XXX bytes
# - Cipher: TLS_AES_256_GCM_SHA384
# - Protocol: TLSv1.3
# - Server Temp Key: X25519, 253 bits
# - TLS session ticket: ...
```

---

## 14. 함정

### 함정 1 — 0-RTT 의 POST
응용이 idempotent 검증 X 시 위험. HTTP/3 의 0-RTT 도 동일.

### 함정 2 — 옛 클라 호환
TLS 1.3 만 지원 시 옛 클라 (Java 7, .NET 4) 깨짐. 1.2 fallback 필요.

### 함정 3 — Middlebox
일부 corporate firewall / IDS 가 TLS 1.3 검사 X. ECH 등 점진 도입.

### 함정 4 — SNI 평문
ECH 도입 전까지 도메인 노출.

### 함정 5 — Session Ticket key
서버 rotation 없으면 모든 옛 세션 위험.

---

## 15. 학습 자료

- **RFC 8446** (TLS 1.3)
- **RFC 8470** (Using Early Data — 0-RTT)
- **RFC 8773** (TLS 1.3 with PSK + Cert)
- Cloudflare blog — TLS 1.3 시리즈
- "Achieving a Big Win for TLS 1.3" — Google

---

## 16. 관련

- [[tls-ssl]] — TLS hub
- [[tls-handshake-1-2]] — 1.2 비교
- [[session-resumption]] — PSK / 0-RTT
- [[../quic/quic-handshake]] — QUIC 도 TLS 1.3
- [[../http/versions/http-3-quic]]
