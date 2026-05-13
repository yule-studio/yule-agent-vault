---
title: "DNS Hierarchy & Resolution"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:25:00+09:00
tags:
  - network
  - dns
  - hierarchy
  - root-server
---

# DNS Hierarchy & Resolution

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Root / TLD / Authoritative / 조회 |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 계층 구조

```
. (Root)
├── com.                  TLD (Top-Level Domain)
│   ├── example.com.      Second-Level Domain (SLD)
│   │   ├── www.          Subdomain
│   │   └── api.
│   └── google.com.
├── kr.                   ccTLD (country code TLD)
│   ├── co.kr.            Second-level (한국 특이)
│   │   └── naver.co.kr.
│   ├── ac.kr.
│   └── go.kr.
├── org.
├── net.
├── io.
├── dev.
└── ...
```

### FQDN — Fully Qualified Domain Name

```
www.example.com.
                ↑ 마지막 점 = root (보통 생략)
```

---

## 2. Root Servers

### 13 개 (이름 a.root-servers.net ~ m.)

| Letter | Operator |
| --- | --- |
| A | Verisign |
| B | USC-ISI |
| C | Cogent |
| D | UMD |
| E | NASA |
| F | ISC |
| G | US DoD |
| H | US Army |
| I | Netnod |
| J | Verisign |
| K | RIPE NCC |
| L | ICANN |
| M | WIDE Project |

### 실제 인스턴스
- 13 "logical" → 수백 "physical" (anycast)
- 전 세계 1000+ 위치
- 사용자 가까운 곳이 응답

자세히 → https://root-servers.org/

---

## 3. TLD 종류

### gTLD (generic)
- .com, .org, .net (옛)
- .info, .biz, .name
- .app, .dev, .io (모던)
- .pizza, .ninja, .blog (newgTLDs)

### ccTLD (country code)
- .kr (한국), .jp, .cn, .uk, .de, .us
- ICANN + 각국 운영

### sponsored TLD
- .gov, .edu, .mil (미국)
- .museum, .aero

### IDN ccTLD
- .한국, .中国, .рф (Cyrillic 등)

---

## 4. Resolution — Recursive vs Iterative

### Recursive Resolver (사용자 입장)

```
Client → Recursive: 답 찾아줘
Recursive: 모든 단계 거쳐 답 가져옴
Recursive → Client: 답
```

→ ISP / Public (8.8.8.8) DNS 가 보통.

### Iterative — Recursive Resolver 의 내부

```
Recursive → Root: "com 의 NS?"
Root → Recursive: "a.gtld-servers.net 등"
Recursive → .com NS: "example.com 의 NS?"
.com NS → Recursive: "ns1.example.com"
Recursive → ns1: "example.com 의 A?"
ns1 → Recursive: "1.2.3.4"
```

→ 한 번에 한 단계씩.

---

## 5. 전체 조회 흐름 — 자세히

### 가정
- 사용자가 `www.example.com` 입력
- 모든 캐시 비어 있음

### Step 1 — Stub Resolver (OS)
```
/etc/resolv.conf:
nameserver 1.1.1.1
nameserver 8.8.8.8
```

OS — `/etc/hosts` 먼저 확인 → 없으면 nameserver 로.

### Step 2 — Recursive 가 Root 에 묻기
```
Recursive → Root server (k.root-servers.net):
  Query: www.example.com (A)
Root → Recursive:
  Authoritative servers for .com:
    a.gtld-servers.net 192.5.6.30
    b.gtld-servers.net ...
    (글루 레코드)
```

### Step 3 — .com NS 에 묻기
```
Recursive → a.gtld-servers.net:
  Query: www.example.com (A)
.com NS → Recursive:
  Authoritative servers for example.com:
    ns1.example.com 1.2.3.10
    ns2.example.com 1.2.3.20
```

### Step 4 — Authoritative 에 묻기
```
Recursive → ns1.example.com:
  Query: www.example.com (A)
ns1 → Recursive:
  www.example.com IN A 1.2.3.4
```

### Step 5 — 캐싱 + 응답
```
Recursive 가 모든 결과 캐시 (TTL 동안)
Recursive → OS resolver: 1.2.3.4
OS → Browser: 1.2.3.4
```

자세히 → [[dns-caching]]

---

## 6. NS 와 SOA Record

### NS (Name Server)
```
example.com.    IN  NS  ns1.example.com.
example.com.    IN  NS  ns2.example.com.
```

→ 이 도메인의 권한 가진 서버.

### SOA (Start of Authority)
```
example.com.    IN  SOA  ns1.example.com. admin.example.com. (
                          2024051401  ; serial
                          3600        ; refresh
                          1800        ; retry
                          1209600     ; expire (14 일)
                          300         ; minimum TTL
                          )
```

→ Zone 의 메타 정보. Secondary NS 가 sync 결정.

자세히 → [[dns-record-types]]

---

## 7. Glue Record

```
ns1.example.com 의 IP 가 필요한데, ns1.example.com 자체가 example.com 안:
"Chicken and egg" 문제

해결: Glue Record
.com NS 가 ns1.example.com 의 A 도 함께 응답
```

→ 같은 도메인의 NS 가 그 도메인 안일 때 필요.

---

## 8. Primary / Secondary NS

### Primary (Master)
- Zone 파일 의 원본
- 변경 가능

### Secondary (Slave)
- Primary 의 copy
- Zone Transfer 로 동기화 (AXFR/IXFR)
- 가용성 ↑

### Hidden Master
- Primary 가 외부 노출 X
- Secondary 만 공개
- 보안 ↑

---

## 9. Anycast

```
같은 IP (8.8.8.8) 를 전 세계 여러 서버가 광고 (BGP)
→ 사용자 가까운 서버로 라우팅
```

### 사용
- Google 8.8.8.8
- Cloudflare 1.1.1.1
- Quad9 9.9.9.9
- Root server (a.root-servers.net 등)

자세히 → [[../osi-7-layer/layer-3-network/multicast-igmp#8 Anycast]]

---

## 10. Resolution 의 측정

### dig 의 query time
```bash
dig example.com
;; Query time: 35 msec
;; SERVER: 1.1.1.1#53
```

### 단계 별
```
Root server (Anycast):    5-20 ms
.com TLD (Anycast):       20-50 ms
Authoritative:             10-100 ms (위치 의존)
캐시 hit:                 < 5 ms
```

→ 첫 조회 (cold) 50-200 ms, 캐시 hit < 10 ms.

---

## 11. Resolution 최적화

### 11.1 짧은 TTL 의 trade-off
- 짧으면 변경 빠름 but 매번 조회
- 길면 캐시 효율 but 변경 늦음

### 11.2 Pre-fetch
- 브라우저가 link 의 host 미리 조회

```html
<link rel="dns-prefetch" href="//other.com">
<link rel="preconnect" href="https://api.com">
```

### 11.3 Multiple Resolver
```
/etc/resolv.conf:
nameserver 1.1.1.1
nameserver 8.8.8.8
nameserver 8.8.4.4
```

→ Primary 실패 시 fallback.

---

## 12. 함정

### 함정 1 — DNS Cache Poisoning
공격자가 잘못된 응답 주입. 방어: DNSSEC, Source Port randomization.

### 함정 2 — DNS Hijacking
ISP / 정부가 DNS 응답 변조. 방어: DoH / DoT.

### 함정 3 — Root 직접 조회
거의 X — Recursive 가 처리. 직접 조회 시 부하.

### 함정 4 — Glue Record 누락
같은 도메인 NS 가 응답 안 됨.

### 함정 5 — Zone Transfer (AXFR) 외부 노출
모든 record 노출 — 정보 누출. ACL.

---

## 13. 학습 자료

- **RFC 1034 / 1035** (DNS)
- **RFC 8482** (HTTPS / SVCB record — 모던)
- **DNS and BIND** (Liu / Albitz)
- ICANN / IANA 문서

---

## 14. 관련

- [[dns]] — DNS hub
- [[dns-record-types]] — NS / SOA / A / ...
- [[dns-caching]] — TTL
- [[dnssec]] — 보안
- [[doh-dot]]
