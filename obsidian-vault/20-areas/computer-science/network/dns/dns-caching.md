---
title: "DNS Caching — TTL / Negative Cache"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:35:00+09:00
tags:
  - network
  - dns
  - caching
  - ttl
---

# DNS Caching — TTL / Negative Cache

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TTL / 캐시 계층 / Negative |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

DNS 응답을 **여러 단계에서 캐시** 해 latency / 부하 ↓. TTL 로 수명 관리.

---

## 2. 캐시 계층

```
Browser cache    (수십 초 - 분)
        ↓
OS cache (stub)  (TTL 까지)
        ↓
Recursive cache  (TTL 까지)
        ↓
Authoritative server
```

### 단계별

| 단계 | 위치 | TTL 따름 |
| --- | --- | --- |
| Browser | Chrome / Firefox 메모리 | 짧게 (1 분 정도) |
| OS | systemd-resolved / dnsmasq | TTL |
| Recursive | ISP / 8.8.8.8 | TTL |
| Authoritative | (원본 — 캐시 X) | — |

---

## 3. TTL — Time to Live

```
example.com.   300   IN  A  1.2.3.4
                ↑ 5 분
```

### 효과
- 응답 받은 시점부터 `now + TTL` 까지 캐시
- 초과 시 origin 에 재조회

### 권장

| 시나리오 | TTL |
| --- | --- |
| 정적 / 변경 없음 | 86400 (1 일) |
| 일반 | 3600 (1 시간) |
| 변경 잦음 | 300 (5 분) |
| Maintenance 전 | 60 (1 분) |
| Cache 무력화 | 0 (캐시 X) |

### TTL trade-off
- 짧음 → 변경 빠름, but 매번 조회 (부하 ↑, latency ↑)
- 김 → 캐시 hit ↑, but 변경 적용 늦음

---

## 4. Maintenance 시 TTL 조정

```
1 주 전: TTL 86400 → 300 (1 일 전 적용 — 옛 TTL 시간)
변경 당일: 새 IP 로 record 변경
모든 캐시 만료 후: TTL 86400 복원
```

→ 짧은 다운타임.

---

## 5. Negative Cache (RFC 2308)

NXDOMAIN / 없음 응답도 캐시:

```
unknown.example.com    NXDOMAIN
→ Recursive 가 캐시 (SOA 의 minimum TTL 동안)
→ 다음 조회 시 즉시 NXDOMAIN
```

### SOA 의 minimum TTL
```
example.com SOA ... 300  ; minimum TTL
                    ↑ 5 분 negative cache
```

### 효과
- "없는 도메인" 의 반복 조회 부담 ↓
- 단점: 도메인 생성 후 잠시 안 보임

---

## 6. Pre-fetch / DNS Prefetch

### HTML
```html
<link rel="dns-prefetch" href="//other.com">
<link rel="preconnect" href="https://api.com">
```

### 효과
- 페이지 로딩 시 미리 DNS 조회
- 실제 요청 시 캐시 hit
- Latency ↓

### preconnect 의 더 강한 효과
- DNS + TCP + TLS 까지

---

## 7. Cache 의 stale / 적용 안 되는 경우

### Cache poisoning
- 공격자가 잘못된 응답 주입
- 방어: DNSSEC, Source Port randomization
- Kaminsky attack (2008)

### TTL 무시
- 일부 OS / 브라우저가 TTL 보다 짧게 캐시
- 예: Chrome 1 분, Firefox 60 분

### Hosts file
- /etc/hosts (Linux/macOS), C:\Windows\System32\drivers\etc\hosts
- DNS 보다 우선 — 디버깅 / 개발 시 활용

---

## 8. DNS Cache Flush

### macOS
```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### Linux (systemd-resolved)
```bash
sudo systemd-resolve --flush-caches
```

### Linux (dnsmasq)
```bash
sudo systemctl restart dnsmasq
```

### Windows
```cmd
ipconfig /flushdns
```

### 브라우저
- Chrome — chrome://net-internals/#dns
- Firefox — about:networking#dns

---

## 9. 분산 환경의 DNS 캐시 일관성

### 문제
```
Region A: DNS 응답 1.2.3.4 (캐시)
Region B: DNS 응답 1.2.3.5 (다른 응답)
```

→ Geo / Latency DNS 에서 의도. 같은 도메인이 다른 응답.

### 해결
- 짧은 TTL
- Health Check + automatic failover
- Anycast (같은 IP)

---

## 10. CDN 의 DNS

### CDN 의 DNS 활용
- Cloudflare / Akamai — DNS 자체가 CDN 의 일부
- 사용자 IP 보고 가장 가까운 edge IP 응답
- Anycast 와 결합

### 단점
- 짧은 TTL (30-60s) — 캐시 효율 ↓
- but 빠른 failover 가능

---

## 11. 함정

### 함정 1 — TTL 너무 짧음
DNS 부하 ↑. 캐시 hit 률 ↓.

### 함정 2 — TTL 너무 김
변경 적용 늦음. Maintenance 시 trade-off.

### 함정 3 — Browser 의 짧은 캐시
TTL 무시. 개발 / 디버깅 시 모를 수 있음.

### 함정 4 — Negative cache 의 stale
도메인 생성 후 잠시 NXDOMAIN. SOA minimum TTL 짧게.

### 함정 5 — Cache poisoning
DNSSEC 권장.

### 함정 6 — /etc/hosts 의 잔여
개발 시 추가한 entry 잊어버림 → 헷갈림.

---

## 12. 학습 자료

- **RFC 1034** Section 4.3.4 (캐시)
- **RFC 2308** (Negative cache)
- "DNS Caching" — Cloudflare

---

## 13. 관련

- [[dns]] — DNS hub
- [[dns-hierarchy-resolution]] — 조회
- [[dns-record-types]] — TTL 의 자리
- [[../http/caching/caching]] — HTTP 캐시와 비교
