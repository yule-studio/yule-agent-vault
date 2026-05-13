---
title: "Cipher Suites — 형식 / AEAD / 권장"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:50:00+09:00
tags:
  - network
  - tls
  - cipher
  - aead
---

# Cipher Suites — 형식 / AEAD / 권장

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLS 1.2/1.3 cipher 형식 + 권장 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. Cipher Suite — 정의

TLS handshake 시 양쪽이 선택하는 **암호 알고리즘 묶음**.

```
키 교환 + 인증 + 대칭 암호 + 무결성 (해시)
```

---

## 2. TLS 1.2 형식

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    │    │    │      │       │       │
    │    │    │      │       │       └─ Hash for HMAC / PRF
    │    │    │      │       └─ AEAD mode
    │    │    │      └─ Symmetric cipher + key size
    │    │    └─ "with"
    │    └─ Authentication (서버 cert 검증 — RSA / ECDSA)
    └─ Key Exchange (ECDHE = Ephemeral ECDH — Forward Secrecy)
```

### 예 분해

| Cipher Suite | 분석 |
| --- | --- |
| `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` | ECDHE / RSA / AES-256-GCM / SHA-384 |
| `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256` | ECDHE / ECDSA / ChaCha20-Poly1305 / SHA-256 |
| `TLS_RSA_WITH_AES_128_CBC_SHA` | RSA / RSA / AES-128-CBC / SHA-1 ❌ |
| `TLS_DHE_RSA_WITH_AES_256_GCM_SHA384` | DHE / RSA / AES-256-GCM / SHA-384 |
| `TLS_RSA_EXPORT_WITH_RC4_40_MD5` | 옛 export 약함 ❌ |

---

## 3. TLS 1.3 형식 (단순화)

```
TLS_AES_256_GCM_SHA384
    │   │       │       │
    │   │       │       └─ Hash for HKDF
    │   │       └─ AEAD
    │   └─ Cipher + key size
    └─ TLS prefix
```

### 5 가지만 (TLS 1.3 표준)

```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

→ Key Exchange / Signature 가 cipher suite 에서 분리 (별도 extension).

---

## 4. 4 가지 구성 요소

### 4.1 Key Exchange

| 알고리즘 | Forward Secrecy | 사용 |
| --- | --- | --- |
| **ECDHE** | ✅ | TLS 1.3 표준 |
| **DHE** | ✅ | TLS 1.2 / 1.3 |
| RSA (key transport) | ❌ | 1.3 제거 |
| PSK | ✅ (with DHE) | IoT / Resumption |
| ~~static DH~~ | ❌ | 사장 |

### 4.2 Authentication (서버 cert)

| 서명 | 키 | 사용 |
| --- | --- | --- |
| **RSA** (with PSS) | 2048-4096 bit | 가장 흔함 |
| **ECDSA** | P-256, P-384 | 모던 — 작은 cert |
| **EdDSA** (Ed25519) | 256 bit | 가장 모던 |
| DSA | 1024 bit | deprecated |

### 4.3 Symmetric Cipher (AEAD 권장)

| 알고리즘 | 종류 | 사용 |
| --- | --- | --- |
| **AES-128-GCM** | AEAD | ✅ |
| **AES-256-GCM** | AEAD | ✅ |
| **ChaCha20-Poly1305** | AEAD | ✅ (mobile 빠름) |
| AES-CCM | AEAD | IoT |
| AES-CBC + HMAC | non-AEAD | TLS 1.2 (deprecated) |
| 3DES | block | 사장 |
| RC4 | stream | 사장 (RFC 7465) |

### 4.4 Hash / MAC

| 알고리즘 | 용도 |
| --- | --- |
| **SHA-256** | HMAC / HKDF / signature |
| **SHA-384** | 더 큰 keys |
| SHA-1 | deprecated |
| MD5 | deprecated |

---

## 5. AEAD vs MAC-then-Encrypt

### AEAD (Authenticated Encryption with Associated Data)
- 한 알고리즘이 암호화 + 무결성
- AES-GCM, ChaCha20-Poly1305
- TLS 1.3 표준

### Mac-then-Encrypt (옛)
```
MAC = HMAC(plaintext, mac_key)
ciphertext = Encrypt(plaintext || MAC, enc_key)
```

→ Padding oracle / 길이 누출 위험 (POODLE, BEAST, Lucky 13).

### Encrypt-then-MAC (RFC 7366)
- TLS 1.2 의 확장
- 안전하지만 사장 (AEAD 표준)

---

## 6. Forward Secrecy

### 정의
서버 private key 도난 후에도 **옛 트래픽 복호화 X**.

### 메커니즘
- 매 세션 새 DH 키 (ephemeral)
- ECDHE / DHE 사용
- RSA key transport 는 FS X (private key 가 곧 모든 옛 세션)

### 중요
- 5 Eyes / NSA 등 국가 감청 우려 후 표준화
- TLS 1.3 강제

---

## 7. 모던 권장 (Mozilla)

### Modern (TLS 1.3 만)

```nginx
ssl_protocols TLSv1.3;
ssl_ciphers TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;
ssl_prefer_server_ciphers off;
```

### Intermediate (1.2 + 1.3)

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

자세히 → https://ssl-config.mozilla.org/

---

## 8. 차단해야 할 Cipher

```
NULL (암호화 X)
EXPORT (옛 미국 export)
RC4 (stream cipher 취약)
3DES (Sweet32)
MD5 (충돌)
SHA-1 (충돌)
DES (40 bit / 56 bit)
PSK without DHE (Forward Secrecy X)
RSA (key exchange, FS X)
CBC (padding oracle 위험 — AEAD 권장)
anonymous (인증 X)
```

### Nginx
```nginx
ssl_ciphers "!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!RC4:..."
```

---

## 9. ECDSA vs RSA Certificate

### RSA
- 가장 호환성 좋음
- 2048-bit 표준, 4096-bit 권장
- 큰 cert (1-2 KB)

### ECDSA
- 더 작음 (256-bit ≈ RSA 3072-bit 보안)
- 더 빠름
- P-256, P-384

### Dual Cert (Hybrid)
```nginx
ssl_certificate rsa.crt;
ssl_certificate_key rsa.key;
ssl_certificate ecdsa.crt;
ssl_certificate_key ecdsa.key;
```

→ 클라가 ECDSA 지원 시 ECDSA, 아니면 RSA.

---

## 10. ChaCha20 vs AES-GCM

### AES-GCM
- HW 가속 (AES-NI) — 데스크톱 빠름
- 모바일 / IoT 의 HW 가속 X — 느림

### ChaCha20-Poly1305
- SW 빠름 (HW 가속 무관)
- 모바일 친화
- Google / Cloudflare 모바일 우선

### 서버의 선택
- 클라가 ChaCha20 우선 명시 → 서버가 따름
- 또는 서버가 클라 분석 후 선택 (BoringSSL의 equal-preference cipher groups)

---

## 11. 도구

```bash
# 클라가 보낸 cipher 목록
openssl s_client -connect example.com:443 -tls1_2 | grep "Cipher is"

# 서버의 지원 cipher 목록
nmap --script ssl-enum-ciphers -p 443 example.com

# testssl.sh
./testssl.sh --protocols --ciphers https://example.com

# SSL Labs
https://www.ssllabs.com/ssltest/
```

---

## 12. 함정

### 함정 1 — 너무 많은 cipher 지원
공격 표면 ↑. Modern / Intermediate 정도.

### 함정 2 — 옛 cipher (RC4, 3DES) 유지
호환성 위해서 — 보안 ↓. 옛 클라 무시.

### 함정 3 — RSA key exchange
Forward Secrecy 없음. ECDHE 만.

### 함정 4 — Cipher 우선순위
TLS 1.2 — `ssl_prefer_server_ciphers on` (서버 우선)
TLS 1.3 — 클라 우선 (off)

### 함정 5 — Ed25519 옛 클라 호환
Java 11+, Go 1.13+ — 옛 stack 은 X. RSA fallback.

---

## 13. 학습 자료

- **RFC 5246** (TLS 1.2), **RFC 8446** (TLS 1.3)
- **RFC 7905** (ChaCha20-Poly1305 in TLS)
- IANA TLS Cipher Suite Registry
- Mozilla SSL Config Generator
- "AEAD" — Phillip Rogaway

---

## 14. 관련

- [[tls-ssl]] — TLS hub
- [[tls-handshake-1-2]] / [[tls-1-3]]
- [[certificates-ca]] — RSA / ECDSA cert
- [[../../../security-theory/security-theory]] — 암호학
