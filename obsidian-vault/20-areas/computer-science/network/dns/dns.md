---
title: "DNS — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:20:00+09:00
tags:
  - network
  - dns
  - domain
---

# DNS — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DNS hub + 6 sub |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-7-application/layer-7-application|↑↑ L7]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L7 Application |
| **포트** | **53** (UDP / TCP), 853 (DoT), 443 (DoH) |
| **표준** | RFC 1034/1035 (1987), 갱신 다수 |
| **출처** | 1983 Paul Mockapetris (USC ISI) |
| **목적** | 도메인 ↔ IP 매핑 + 메타데이터 |

---

## 1. 한 줄 정의

**D**omain **N**ame **S**ystem — 사람이 읽는 도메인 (`example.com`) 을 IP 주소
(`1.2.3.4`) 로 변환. 인터넷의 전화번호부. 분산 / 계층 / 캐시.

---

## 2. 왜 DNS 인가

```
사용자: example.com
컴퓨터: ???

→ DNS 가 example.com → 1.2.3.4 변환
→ TCP 연결 시작
```

### 변환 외 정보
- 메일 서버 (MX)
- 도메인 인증 (TXT — SPF, DKIM, DMARC)
- 보안 (CAA, DNSSEC)
- 서비스 발견 (SRV)
- 모던 — HTTPS / SVCB (HTTP/3 발견)

---

## 3. 계층 구조

```
.                        (Root)
├── com.                 (TLD)
│   ├── example.com.     (Domain)
│   │   ├── www.
│   │   ├── api.
│   │   └── mail.
│   └── google.com.
├── org.
├── kr.                  (Country code TLD)
│   ├── co.kr.
│   │   └── naver.co.kr.
│   └── ac.kr.
└── net.
```

→ 점 (`.`) 으로 구분, 오른쪽에서 왼쪽으로 더 일반.

---

## 4. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[dns-hierarchy-resolution]] | 계층 + 조회 흐름 (Root → TLD → Authoritative) |
| [[dns-record-types]] | A / AAAA / CNAME / MX / TXT / NS / SOA / SRV / PTR / CAA / HTTPS |
| [[dns-caching]] | TTL / 재귀 / 권한 / Negative cache |
| [[dnssec]] | DNS 서명 / 인증 |
| [[doh-dot]] | DNS over HTTPS / DNS over TLS / 프라이버시 |
| [[dns-tools]] | dig / nslookup / host / dnsperf |

---

## 5. DNS 서버의 종류

| 종류 | 역할 | 예 |
| --- | --- | --- |
| **Stub Resolver** | 클라이언트 (OS) | `/etc/resolv.conf` |
| **Recursive Resolver** | 사용자 대신 조회 | ISP DNS, 8.8.8.8, 1.1.1.1 |
| **Authoritative** | 도메인 zone 의 권한 보유 | route 53, Cloudflare DNS |
| **Forwarder** | 다른 resolver 로 전달 | 회사 내부 |
| **Root** | 13 root server (Anycast) | a.root-servers.net 등 |

자세히 → [[dns-hierarchy-resolution]]

---

## 6. 조회 흐름 — "example.com" 처음 방문

```
1. 브라우저 → OS resolver: example.com?
2. OS resolver → /etc/hosts 확인 → 없음
3. OS resolver → Recursive (1.1.1.1)
4. Recursive: 캐시 확인 → 없음
5. Recursive → Root server: ".com" 의 NS?
6. Root → Recursive: .com 의 NS (a.gtld-servers.net)
7. Recursive → .com NS: "example.com" 의 NS?
8. .com NS → Recursive: example.com 의 NS (ns1.example.com)
9. Recursive → ns1.example.com: example.com 의 A?
10. ns1.example.com → Recursive: 1.2.3.4
11. Recursive → OS: 1.2.3.4
12. OS → 브라우저: 1.2.3.4
```

→ 보통 50-200 ms. 캐시 hit 시 < 10 ms.

---

## 7. UDP vs TCP

### 기본 — UDP
- 작은 query / 응답 (< 512 byte)
- 빠름

### TCP — 큰 응답
- 응답 > 512 byte → TC (Truncated) flag
- 클라가 TCP 로 재시도
- DNSSEC / 큰 응답 흔함

### EDNS0 (RFC 6891)
- UDP buffer 확장 (최대 4096 byte)
- 큰 응답도 UDP 가능

### DoH / DoT
- DNS over HTTPS / TLS — 항상 TCP

자세히 → [[doh-dot]]

---

## 8. 도구

```bash
# dig — 가장 강력
dig example.com
dig example.com A +short
dig example.com MX
dig example.com NS
dig @8.8.8.8 example.com
dig example.com +trace        # Root 부터 추적

# nslookup
nslookup example.com
nslookup -type=MX example.com

# host
host example.com
host -t MX example.com

# 응답 시간 측정
dig example.com | grep "Query time"
```

자세히 → [[dns-tools]]

---

## 9. 함정

### 함정 1 — TTL 잘못
짧으면 cache hit 률 ↓, 길면 변경 적용 늦음. 보통 300-3600 초.

### 함정 2 — 잘못된 NS 설정
도메인의 NS 가 잘못 — 권한 잃음.

### 함정 3 — CNAME 의 root domain
`@ CNAME other.com` — RFC 위반 (root 에 CNAME X). ALIAS / ANAME 사용.

### 함정 4 — DNS 캐시의 stale
TTL 후에도 stale 응답 사용. CDN / proxy 의 캐시 정책.

### 함정 5 — DNSSEC 의 운영 복잡성
잘못된 서명 → 도메인 자체 안 보임.

### 함정 6 — 도메인 만료
WHOIS 등록 정보 갱신 누락.

---

## 10. 면접 / 토픽

1. **DNS 조회 흐름** (Root → TLD → Authoritative).
2. **Recursive vs Authoritative**.
3. **A vs AAAA vs CNAME**.
4. **TTL 의 의미**.
5. **DNSSEC** — 무엇 / 어떻게.
6. **DoH / DoT** — 왜.
7. **DNS의 UDP vs TCP**.

---

## 11. 학습 자료

- **RFC 1034 / 1035** (DNS 기본)
- **DNS and BIND** (Liu / Albitz)
- "DNS for Rocket Scientists" — Pro DNS
- Cloudflare blog — DNS

---

## 12. 관련

- [[../osi-7-layer/layer-7-application/layer-7-application]] — L7
- [[../tls-ssl/tls-ssl]] — HTTPS DNS Record (모던)
- [[dns-hierarchy-resolution]] / [[dns-record-types]] / [[dns-caching]] / [[dnssec]] / [[doh-dot]] / [[dns-tools]]
