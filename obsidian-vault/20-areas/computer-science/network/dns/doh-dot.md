---
title: "DNS over HTTPS (DoH) / DNS over TLS (DoT) / DoQ"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:45:00+09:00
tags:
  - network
  - dns
  - doh
  - dot
  - privacy
---

# DNS over HTTPS (DoH) / DNS over TLS (DoT) / DoQ

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DoH / DoT / DoQ 비교 + 프라이버시 |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

DNS 쿼리를 **암호화** 해 도청 / 변조 / 검열 방지. 평문 DNS (UDP 53) 의 모던 대체.

- **DoT** (DNS over TLS) — RFC 7858, 2016, 포트 853
- **DoH** (DNS over HTTPS) — RFC 8484, 2018, 포트 443
- **DoQ** (DNS over QUIC) — RFC 9250, 2022, 포트 853

---

## 2. 평문 DNS 의 문제

### 도청
```
공항 Wi-Fi 에서 사용자의 모든 DNS 쿼리 보임
→ 어느 사이트 방문하는지 모두 노출
```

### 변조 (Hijacking)
- ISP 가 광고 페이지로 redirect
- 정부가 차단 (DNS-based censorship)

### Cache Poisoning
- Spoofed 응답

→ DNSSEC 는 무결성만, **기밀성 X**.

---

## 3. DoT — DNS over TLS

### 흐름
```
Client → Recursive: TCP 853 + TLS handshake
Client → Recursive: DNS query (TLS 안)
Recursive → Client: DNS response (TLS 안)
```

### 특징
- 별도 포트 (853) — 식별 쉬움
- TLS handshake 비용 (재사용으로 줄임)
- 방화벽 / DNS 차단 시 막을 수 있음 (포트 명확)

---

## 4. DoH — DNS over HTTPS

### 흐름
```
Client → Resolver: HTTPS POST /dns-query (또는 GET ?dns=)
              Content-Type: application/dns-message
              Body: DNS 메시지 (binary)
Resolver → Client: HTTPS response with DNS message
```

### 특징
- **HTTPS 트래픽과 구분 X** (포트 443)
- 차단 어려움 (HTTPS 전체 차단해야)
- 검열 회피 / 프라이버시 좋음
- HTTP/2 멀티플렉싱 활용

---

## 5. DoQ — DNS over QUIC

### 특징
- QUIC (UDP 443) 위
- 0-RTT 가능
- HTTP/3 의 transport 와 같음

### 도입
- 2022 표준화
- 일부 Public DNS 지원

---

## 6. Public DoH / DoT 서비스

| 서비스 | DoT | DoH | IP |
| --- | --- | --- | --- |
| **Cloudflare** | tls://1.1.1.1 | https://cloudflare-dns.com/dns-query | 1.1.1.1 |
| **Google** | tls://dns.google | https://dns.google/dns-query | 8.8.8.8 |
| **Quad9** | tls://dns.quad9.net | https://dns.quad9.net/dns-query | 9.9.9.9 |
| **AdGuard** | tls://dns.adguard.com | https://dns.adguard.com/dns-query | 94.140.14.14 |
| **NextDNS** | (Custom) | (Custom) | — |

### 특징
- **Cloudflare** — 가장 빠름 (Anycast)
- **Quad9** — 멀웨어 차단
- **AdGuard** — 광고 차단 + DNS

---

## 7. 클라 측 설정

### macOS / Windows / iOS / Android
- OS 설정 — Private DNS / Secure DNS
- Android 9+ Private DNS (DoT)
- iOS 14+ Encrypted DNS

### 브라우저
- Chrome — Settings → Privacy → Use secure DNS
- Firefox — Network Settings → Enable DNS over HTTPS

### Linux
```bash
# systemd-resolved (DoT)
/etc/systemd/resolved.conf:
DNS=1.1.1.1#cloudflare-dns.com
DNSOverTLS=yes
```

### 응용 라이브러리
- doh-client (DNS proxy)
- Cloudflare cloudflared
- dnscrypt-proxy

---

## 8. 서버 측 — DoH/DoT 운영

### Cloudflare DNS server
- 자체 운영
- Anycast 의 일부

### 자체 운영
- **knot-resolver** — DoT/DoH 지원
- **unbound** — DoT
- **CoreDNS** — DoH/DoT plugin

### 인증서
- Let's Encrypt 등 일반 TLS cert

---

## 9. 비교 — DoH vs DoT vs 평문

| 측면 | 평문 53 | DoT 853 | DoH 443 | DoQ 443 (UDP) |
| --- | --- | --- | --- | --- |
| Transport | UDP/TCP | TCP+TLS | TCP+TLS | UDP+QUIC |
| 암호화 | ❌ | ✅ | ✅ | ✅ |
| 식별 | 쉬움 | 쉬움 (853) | 어려움 | 어려움 |
| 차단 | 쉬움 | 쉬움 | 어려움 (HTTPS 와 같음) | 어려움 |
| 0-RTT | — | 일부 | ❌ | ✅ |
| 성능 | 가장 빠름 | 느림 (TLS) | 더 느림 (HTTP) | 빠름 (QUIC) |

→ 모던 권장: **DoH** (검열 회피) 또는 **DoQ** (속도+프라이버시).

---

## 10. ODoH — Oblivious DoH (RFC 9230)

### 동기
- DoH 도 resolver 가 (사용자 IP, query) 모두 알음
- 프라이버시 100% X

### ODoH 구조
```
Client → Proxy: 암호화된 query
Proxy → Target Resolver: 암호화 query forward
Target Resolver: 복호화 → 응답 → Proxy
Proxy → Client: 응답
```

- Proxy 는 Client 의 IP 알지만 query 모름
- Target Resolver 는 query 알지만 누구 (Proxy IP) 모름
- 분리로 추적 X

### 도입
- Cloudflare + Apple (iCloud Private Relay 의 일부)

---

## 11. Browser 의 DoH 정책

### Firefox (2020+)
- Default — Cloudflare DoH (북미)
- TRR (Trusted Recursive Resolver)
- 사용자 동의 후 활성

### Chrome
- Secure DNS 옵션
- 자동 — 같은 resolver provider 의 DoH 사용

### Safari (iOS 14+)
- iCloud Private Relay (ODoH 같은 효과)

---

## 12. 함정

### 함정 1 — 회사 / 국가 차단
- DoH 의 호스트 (cloudflare-dns.com) DNS 차단
- 한국 일부 ISP — 광고 차단 DNS 차단

### 함정 2 — Public DNS 의 신뢰
- Cloudflare / Google 도 데이터 수집 가능
- "No logs" 정책 — 신뢰 의존

### 함정 3 — Captive Portal
- Wi-Fi 의 로그인 페이지가 DNS hijack
- DoH 만 쓰면 detect 어려움 — fallback 필요

### 함정 4 — 응용의 DNS bypass
- 일부 응용이 OS DNS 무시 — DoH bypass

### 함정 5 — DNSSEC + DoH
- 둘 다 권장
- DoH = 기밀성, DNSSEC = 무결성

### 함정 6 — Anti-malware DNS
- 회사 / 학교의 DNS 기반 차단 — DoH 로 우회됨
- Enterprise 가 DoH 차단 시도

---

## 13. 측정 / 도구

```bash
# kdig (DoT/DoH)
kdig +tls @1.1.1.1 example.com
kdig +https @1.1.1.1 example.com

# curl 로 DoH
curl -H 'accept: application/dns-message' \
  'https://cloudflare-dns.com/dns-query?dns=...'

# 또는 GET wireformat
curl -H 'accept: application/dns-json' \
  'https://cloudflare-dns.com/dns-query?name=example.com&type=A'
```

---

## 14. 학습 자료

- **RFC 7858** (DoT), **RFC 8484** (DoH), **RFC 9250** (DoQ), **RFC 9230** (ODoH)
- "Encrypted DNS" — Cloudflare
- Mozilla TRR 가이드
- "DNS over HTTPS" — IETF

---

## 15. 관련

- [[dns]] — DNS hub
- [[dnssec]] — 무결성 (보완 관계)
- [[../tls-ssl/tls-ssl]] — TLS
- [[../quic/quic]] — DoQ
- [[../topics/topics]] — Zero Trust
