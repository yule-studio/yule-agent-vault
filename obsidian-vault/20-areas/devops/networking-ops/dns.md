---
title: "DNS — Route53 / Cloudflare / 권한위임"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:52:00+09:00
tags: [devops, networking-ops, dns]
---

# DNS — Route53 / Cloudflare / 권한위임

**[[networking-ops|↑ networking-ops]]**

---

## 1. DNS 흐름

```
[client] → [resolver (ISP, 8.8.8.8)] → [root] → [TLD (.com)] → [authoritative NS] → IP
                  ↓
                  cache (TTL)
```

→ resolver 가 cache 하므로 변경 즉시 반영 X.

---

## 2. record 종류

| Type | 무엇 | 예 |
| --- | --- | --- |
| **A** | IPv4 | `example.com → 1.2.3.4` |
| **AAAA** | IPv6 | `example.com → 2001:...` |
| **CNAME** | alias | `www.example.com → example.com` |
| **MX** | mail | `example.com → 10 mail.example.com` |
| **TXT** | text | SPF / DKIM / verification |
| **NS** | name server | `example.com → ns1.cloudflare.com` |
| **SOA** | zone info | (auto) |
| **PTR** | reverse | `4.3.2.1.in-addr.arpa → example.com` |
| **CAA** | 인증서 발급 권한 | `example.com 0 issue "letsencrypt.org"` |
| **SRV** | service location | `_sip._tcp.example.com → ...` |
| **ALIAS / ANAME** | apex CNAME (Route53/Cloudflare 만) | `example.com → alb.aws.com` |

---

## 3. Route53 routing 정책

| 정책 | 무엇 | 사용 |
| --- | --- | --- |
| **simple** | 1 record | 일반 |
| **weighted** | 가중치 분배 | A/B test, blue-green |
| **latency-based** | 가까운 region | global |
| **failover** | primary fail 시 secondary | DR |
| **geolocation** | 국가별 다른 IP | 컴플라이언스 |
| **geoproximity** | 지리적 거리 + bias | edge balance |
| **multi-value** | 여러 IP, health check | basic LB |

---

## 4. failover 예 (Route53)

```yaml
# primary
- type: A
  name: api.example.com
  routing: failover
  set-identifier: primary
  alias: alb-us-east-1
  health-check: us-east-1-health

# secondary
- type: A
  name: api.example.com
  routing: failover
  set-identifier: secondary
  alias: alb-us-west-2
```

→ us-east-1 down → 자동 us-west-2.

---

## 5. TTL

```
짧음 (60s)   — 빠른 변경, 더 많은 resolver query
긺 (1 day)   — cache hit ↑, 변경 늦음

표준:
  A / AAAA: 300s (5min)
  MX / TXT: 3600s (1h)
  failover: 60s
  변경 직전: 60s 로 미리 줄이기 → 변경 → 다시 늘림
```

---

## 6. 검증

```bash
dig example.com
dig example.com +trace          # 단계별
dig @8.8.8.8 example.com        # 특정 resolver
dig example.com MX +short
dig +short TXT _dmarc.example.com

# propagation 확인 (전세계)
https://www.whatsmydns.net/

# DNSSEC
dig example.com +dnssec
```

---

## 7. SPF / DKIM / DMARC (메일)

```
# SPF (TXT)
v=spf1 include:_spf.google.com include:amazonses.com -all

# DKIM (TXT 서브도메인)
selector._domainkey.example.com:
v=DKIM1; k=rsa; p=MIGfMA...

# DMARC (TXT)
_dmarc.example.com:
v=DMARC1; p=reject; rua=mailto:dmarc@example.com
```

→ [[../../60-recipes/spring/notification/notification|notification recipe]] 의 이메일 발송 시 필수.

---

## 8. apex (root) domain 의 CNAME 문제

```
DNS 표준: apex 에 CNAME ❌ (NS / SOA 와 충돌)

해결:
- Route53 의 Alias (CNAME 처럼 보이지만 A record)
- Cloudflare 의 CNAME flattening
- 또는 www.example.com 만 사용 + apex → 301 redirect
```

---

## 9. CAA record (★ TLS 보안)

```
example.com.  CAA 0 issue "letsencrypt.org"
example.com.  CAA 0 issue "amazon.com"
```

→ Let's Encrypt + AWS ACM 만 cert 발급 가능. 다른 CA 가 발급 시도 = 거부.

→ 안 하면 누군가 다른 CA 로 phishing cert 발급 가능.

---

## 10. private DNS (k8s, VPC)

```
k8s: <service>.<ns>.svc.cluster.local
     CoreDNS 가 resolve

AWS: VPC 의 .internal / Route53 Private Hosted Zone
GCP: Cloud DNS Private
```

---

## 11. DNSSEC

```
DNS 응답에 서명 → 위조 방지.

설정:
1. registrar 에서 DS record 등록
2. authoritative NS 에서 signing 활성

Route53 / Cloudflare 가 자동.
```

→ 일반 사용자 효과는 작지만 (대부분 resolver 검증 X), 정부 / 금융 권장.

---

## 12. 함정

1. **TTL 너무 김** — 변경 후 며칠 wait.
2. **변경 전 TTL 줄이기 안 함** — 사용자 영향.
3. **apex CNAME** — RFC 위반.
4. **propagation 1분 만에 끝나는 줄** — 24-48h 까지 걸림.
5. **CAA 없음** — phishing cert.
6. **glue record 없음** — `ns1.example.com` 자기 자신 lookup (chicken-egg).
7. **NS 변경 후 옛 NS 에 record 남음** — 일부 사용자 옛 IP.

---

## 13. 관련

- [[networking-ops|↑ networking-ops]]
- [[../linux/networking-basics|↗ linux dig]]
- [[ssl-tls-ops]]
- [[../../60-recipes/spring/notification/notification|↗ 이메일 SPF/DKIM]]
