---
title: "TLS / SSL 역사 + Deprecated 버전 + 주요 공격"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T02:35:00+09:00
tags:
  - network
  - tls
  - ssl
  - history
  - poodle
---

# TLS / SSL 역사 + Deprecated 버전 + 주요 공격

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SSL → TLS 진화 + 공격 |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 시대순 진화

| 연도 | 사건 |
| --- | --- |
| 1994 | Netscape — SSL 1.0 (비공개, 결함) |
| 1995 | SSL 2.0 — 첫 공개 |
| 1996 | SSL 3.0 — Netscape |
| 1999 | **TLS 1.0** (RFC 2246) — IETF 표준 |
| 2006 | TLS 1.1 (RFC 4346) |
| 2008 | **TLS 1.2** (RFC 5246) |
| 2014 | POODLE — SSL 3.0 차단 |
| 2018 | **TLS 1.3** (RFC 8446) |
| 2020 | 주요 브라우저 TLS 1.0/1.1 차단 |
| 2021 | RFC 8996 — TLS 1.0/1.1 deprecated |

---

## 2. SSL 1.0 — 비공개 (1994)

- Netscape 내부 만
- 보안 결함으로 공개 X
- 역사적 의의만

---

## 3. SSL 2.0 (1995) — RFC 6176 금지

### 문제
- MAC 와 키가 분리 X (단일 MD5 hash)
- Padding oracle 가능
- ChangeCipherSpec 의 buffer overflow
- Cipher Suite downgrade 공격 가능

### 결과
- RFC 6176 (2011) — "절대 사용 X"
- 모든 브라우저 / 라이브러리 제거

---

## 4. SSL 3.0 (1996) — POODLE 후 차단

### POODLE (2014) 공격
- **P**adding **O**racle **O**n **D**owngraded **L**egacy **E**ncryption
- CBC mode 의 padding 검증 oracle
- 1 byte 씩 평문 추출

### 동작
```
1. 공격자 (MITM): 클라가 SSL 3.0 으로 fallback 강제
2. 암호화된 cookie 의 byte 별 추측
3. Padding 오류로 정답 / 오답 구분
```

### 방어
- TLS_FALLBACK_SCSV (자동 downgrade 차단)
- SSL 3.0 완전 제거

---

## 5. TLS 1.0 (1999) — RFC 8996 폐기

### 의의
- SSL 3.0 의 IETF 표준화
- IV 처리, MAC 등 약점 개선

### 공격

#### BEAST (2011)
- **B**rowser **E**xploit **A**gainst **S**SL/**T**LS
- CBC mode 의 predictable IV
- TLS 1.0 만 영향, TLS 1.1+ OK

#### Lucky 13 (2013)
- CBC 의 timing attack
- 패치 가능

### 폐기
- 2020 Chrome / Firefox / Safari 차단
- 2021 RFC 8996 — 공식 폐기

---

## 6. TLS 1.1 (2006) — RFC 8996 폐기

### 개선
- IV 명시적 (BEAST 방어)
- Padding 오류 정보 누출 제거

### 한계
- 1.2 의 AEAD / SHA-256 없음
- 1.1 만의 차이 점은 미미

### 폐기
- TLS 1.0 과 함께 차단 (2020)

---

## 7. TLS 1.2 (2008) — 현역

### 개선
- **AEAD** 지원 (AES-GCM, ChaCha20-Poly1305)
- SHA-256 등 모던 hash
- Cipher Suite 명시적 선택
- Renegotiation 보안 강화 (RFC 5746)

### 한계 — TLS 1.3 의 동기
- 2-RTT handshake
- RSA key exchange (Forward Secrecy X)
- 약한 cipher 여전히 (3DES, RC4 — deprecated 지만 남음)
- Cipher Suite 폭증 (300+)

---

## 8. TLS 1.3 (2018) — 모던

자세히 → [[tls-1-3]]

### 혁신
- 1-RTT (재방문 0-RTT)
- **Forward Secrecy 강제** ((EC)DHE 만)
- AEAD 만 (5 cipher suite 만)
- 모든 deprecated 제거

---

## 9. 주요 공격 / 사고 종합

### 9.1 Heartbleed (2014, CVE-2014-0160)

#### 동작
- OpenSSL 의 Heartbeat extension 의 buffer over-read
- 64 KB 의 서버 메모리 → 비밀 (private key, password) 노출

#### 영향
- 인터넷의 약 17% 서버
- 모든 cert 회전 + OpenSSL 패치
- 보안 사상 최악 중 하나

#### 방어
- OpenSSL 1.0.1g+
- 영향 cert 모두 회전

### 9.2 FREAK (2015)

- **F**actoring **R**SA **E**xport **K**eys
- 옛 export-grade RSA (512 bit) 강제
- 클라가 export cipher 받음 → RSA 1024 약함

#### 방어
- 모든 export cipher 제거

### 9.3 Logjam (2015)

- DH key exchange 약한 그룹 (1024 bit)
- 사전 계산으로 NSA 같은 국가가 깰 수 있음

#### 방어
- DH 2048+ 또는 ECDHE
- 약한 export DH 제거

### 9.4 DROWN (2016)

- **D**ecrypting **R**SA with **O**bsolete and **W**eakened e**N**cryption
- SSLv2 가 다른 곳 운영 시 — TLS 의 RSA 키 노출 (cross-protocol)

#### 방어
- SSLv2 완전 제거 (어떤 서비스든)

### 9.5 ROBOT (2017)

- **R**eturn **O**f **B**leichenbacher's **O**racle **T**hreat
- 1998 Bleichenbacher 공격의 재발견
- RSA PKCS#1 v1.5 padding oracle
- Facebook / PayPal 등 영향

#### 방어
- RSA key exchange 제거 (TLS 1.3)
- OAEP padding

### 9.6 CRIME (2012)

- **C**ompression **R**atio **I**nfo-leak **M**ade **E**asy
- TLS compression 의 압축률로 비밀 추측
- 1 byte 씩 cookie 추측

#### 방어
- TLS compression 제거 (TLS 1.3)

### 9.7 BREACH (2013)

- HTTP compression + TLS 의 같은 원리
- HTTPS 응답이 압축 + 사용자 입력 반영 시

#### 방어
- 비밀 응답 = compression 끄기
- 사용자 입력 secret 분리

### 9.8 Sweet32 (2016)

- 64-bit block cipher (3DES, Blowfish) 의 birthday attack
- 32 GB+ 트래픽 시 키 추출

#### 방어
- 3DES 제거 — AES 만

---

## 10. 모던 TLS Configuration 권장

### Mozilla Server Side TLS

#### Modern (TLS 1.3 만)
```
Protocols: TLS 1.3
Cipher Suites:
  TLS_AES_128_GCM_SHA256
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256
```

#### Intermediate (TLS 1.2 + 1.3)
```
Protocols: TLS 1.2, TLS 1.3
Cipher Suites:
  TLS_AES_*_GCM_*
  TLS_CHACHA20_POLY1305_*
  ECDHE-ECDSA-AES128-GCM-SHA256
  ECDHE-ECDSA-AES256-GCM-SHA384
  ECDHE-RSA-AES128-GCM-SHA256
  ECDHE-RSA-AES256-GCM-SHA384
  ECDHE-ECDSA-CHACHA20-POLY1305
  ECDHE-RSA-CHACHA20-POLY1305
```

#### Old (호환성)
- 옛 클라 호환 필요 시 — 대부분 불필요

### 도구
- Mozilla SSL Config Generator — https://ssl-config.mozilla.org/

---

## 11. SSL Labs 등급

```
A+: 모든 모범 사례 (HSTS, Forward Secrecy, ...)
A:  현역
B:  취약점 일부
C:  심각 취약
F:  보안 X (SSL 2.0 등)
```

→ https://www.ssllabs.com/ssltest/

---

## 12. 학습 자료

- **Bulletproof SSL and TLS** (Ivan Ristić)
- "POODLE Attack" — Google 발표
- "Heartbleed" — Codenomicon
- Cloudflare blog — TLS 시리즈

---

## 13. 관련

- [[tls-ssl]] — TLS hub
- [[tls-handshake-1-2]] / [[tls-1-3]]
- [[cipher-suites]] — 권장 / 차단
- [[../../../security-theory/security-theory]] — 암호학
