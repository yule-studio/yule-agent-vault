---
title: "DNSSEC — DNS Security Extensions"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:40:00+09:00
tags:
  - network
  - dns
  - dnssec
  - security
---

# DNSSEC — DNS Security Extensions

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DNSKEY / RRSIG / DS / NSEC |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

DNS 응답에 **디지털 서명** 을 추가해 위변조 감지. RFC 4033-4035. **암호화 X — 무결성만**.

---

## 2. 동기

### DNS 의 취약점
- UDP 기반 — Spoofing 쉬움
- Source Port 추측 → cache poisoning (Kaminsky 2008)
- ISP / 정부의 DNS 변조

### DNSSEC 의 보장
- **Origin authentication** — 답이 정말 그 도메인의 NS 에서
- **Data integrity** — 변조 X
- **Negative response** — "없다" 도 서명

→ 기밀성 X (그건 DoH/DoT).

---

## 3. 핵심 Record

### DNSKEY — Zone 의 공개키

```
example.com.   IN  DNSKEY  257 3 13  <Base64 public key>
                            │   │  │
                            │   │  └─ 알고리즘 (13 = ECDSAP256SHA256)
                            │   └─ 프로토콜 (3)
                            └─ Flags (257 = KSK, 256 = ZSK)
```

### KSK — Key Signing Key
- 다른 DNSKEY 만 서명
- 부모 zone 의 DS 에 등록
- 회전 어려움

### ZSK — Zone Signing Key
- Record 들 서명
- 자주 회전 (3-6 개월)

---

### RRSIG — Resource Record Signature

```
example.com.   IN  A      1.2.3.4
example.com.   IN  RRSIG  A 13 2 300 ...  <서명>
                            │  │  │  └ 만료 등
                            │  │  └─ TTL
                            │  └─ Labels
                            └─ 알고리즘
```

### 의미
- 각 RRset (같은 이름 + type) 마다 서명
- ZSK 로 서명
- 클라가 DNSKEY 로 검증

---

### DS — Delegation Signer

부모 zone (예: `.com`) 에 자식 zone (`example.com`) 의 KSK hash:

```
example.com.   IN  DS  <key tag> 13 2  <hash of DNSKEY>
                       │           │   │
                       │           │   └─ Digest (SHA-256)
                       │           └─ Digest Algorithm
                       └─ Key Tag
```

### 의미
- 부모가 자식의 키를 "이게 진짜" 라고 보증
- 신뢰 체인의 핵심

---

### NSEC / NSEC3 — Authenticated Denial

```
NSEC 만 — "이 도메인 다음은 X" — 모든 도메인 알 수 있음 (zone walking)
NSEC3 — Hash 로 무작위화 — zone walking 방어
```

### 사용
- "이 record 없다" 의 증명
- 서명된 negative response

---

## 4. Chain of Trust

```
. (Root)
KSK signs DNSKEYs
       ↓ DS in .com (parent)
.com
KSK signs DNSKEYs
       ↓ DS in example.com (parent)
example.com
KSK signs DNSKEYs
       ↓
ZSK signs all records (A, MX, ...)
```

### Root Key
- IANA 가 관리 (KSK)
- 공개 (ICANN Trust Anchor)
- 클라 / Recursive 가 hardcoded

### 검증
```
Recursive 가 응답 받음:
1. RRSIG 의 ZSK 로 검증
2. ZSK 가 KSK 서명? 검증
3. KSK 의 hash 가 부모의 DS 와 일치?
4. 부모의 DS 도 같은 방식으로 ... → Root 까지
5. Root key 의 신뢰 (hardcoded)
```

---

## 5. 활성화 흐름

### 도메인 측
1. 도메인의 nameserver 가 DNSSEC 지원
2. KSK / ZSK 생성
3. 모든 record 에 RRSIG
4. DNSKEY publish
5. 부모 zone (registrar) 에 DS 등록

### Resolver 측
- Root key 신뢰
- 모든 응답 자동 검증

### Cloudflare / Route 53 등
- 자동 — UI 에서 토글
- KSK / ZSK 자동 관리 + 회전
- 부모 zone (registrar) 에 DS 등록만 수동

---

## 6. DNSSEC 의 한계

### 6.1 기밀성 X
- 응답은 평문 — DoH / DoT 가 보완

### 6.2 응답 크기 ↑
- 서명 추가로 큰 응답
- UDP 단편화 — TCP fallback
- EDNS0 4096 byte

### 6.3 운영 복잡성
- 잘못된 서명 → 도메인 자체 안 보임 (SERVFAIL)
- 만료 → 같은 결과
- 자동화 (cert-manager 등) 권장

### 6.4 도입률
- Authoritative: ~30% (점진)
- Recursive 검증: 65%+ (Cloudflare 1.1.1.1, 8.8.8.8 활성)
- 사용자가 DNSSEC 검증 알기 어려움

### 6.5 옛 클라
- Hardware / OS 가 모르면 무시
- 검증 안 함 — 보안 없음

---

## 7. DANE — DNS-based Authentication of Named Entities

### TLSA Record
```
_443._tcp.example.com.   IN  TLSA  3 1 1  <hash of public key>
                                    │ │ │
                                    │ │ └─ Matching Type (SHA-256)
                                    │ └─ Selector (SubjectPublicKeyInfo)
                                    └─ Usage (DANE-EE)
```

### 의미
- TLS cert 의 hash 를 DNS 에 (DNSSEC 보호)
- 클라가 TLS handshake 후 검증
- CA 의존 ↓ (CA 가 잘못 발급해도 무시)

### 한계
- 거의 사용 X — 브라우저 비활성
- 메일 SMTP MTA-STS / DANE 일부

---

## 8. DDoS Amplification

### 문제
- DNSSEC 응답이 큼 — UDP amplification 공격 도구화
- 작은 요청 + 큰 응답 → 피해자에 폭격

### 방어
- DNS RRL (Response Rate Limiting)
- BCP 38 (Source IP filtering)
- TCP fallback

---

## 9. 디버깅

### dig +dnssec
```bash
dig +dnssec example.com

# 출력:
# ;; flags: qr rd ra ad     ← "ad" = Authenticated Data
# example.com.  300  IN  A     1.2.3.4
# example.com.  300  IN  RRSIG A 13 2 300 ...

# DNSSEC chain 추적
dig +trace +dnssec example.com
```

### Verisign DNSSEC Debugger
- https://dnssec-debugger.verisignlabs.com/example.com

### Cloudflare 1.1.1.1
- DNSSEC validation 자동
- 실패 시 SERVFAIL

---

## 10. 함정

### 함정 1 — 잘못된 서명
도메인 자체 안 보임. 모니터링 + 자동 갱신.

### 함정 2 — DS 미등록
부모 zone 에 DS 없으면 — DNSSEC 비활성 효과.

### 함정 3 — KSK 회전
복잡 — 부모 zone 협조 필요. Algorithm rollover 어려움.

### 함정 4 — Negative response
NSEC zone walking 으로 모든 도메인 노출. NSEC3 권장.

### 함정 5 — 응답 크기 폭증
TCP fallback 의존. EDNS0 활성.

### 함정 6 — Lookaside Validation (DLV) — deprecated
옛 — 부모 zone 이 DNSSEC 안 할 때 우회. 사장.

---

## 11. 학습 자료

- **RFC 4033 / 4034 / 4035** (DNSSEC)
- **RFC 5155** (NSEC3)
- **RFC 6698** (DANE)
- "DNSSEC HOWTO" — NLnet Labs
- Cloudflare DNSSEC 가이드

---

## 12. 관련

- [[dns]] — DNS hub
- [[dns-record-types]] — DNSKEY/RRSIG/DS/NSEC
- [[doh-dot]] — 기밀성 (DNSSEC 의 보완)
- [[../../../security-theory/security-theory]] — PKI / 디지털 서명
