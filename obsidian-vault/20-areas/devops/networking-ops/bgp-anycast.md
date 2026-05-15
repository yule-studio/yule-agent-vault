---
title: "BGP / anycast / global routing ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:45:00+09:00
tags: [devops, networking-ops, bgp, anycast]
---

# BGP / anycast / global routing ★

**[[networking-ops|↑ networking-ops]]**

---

## 1. BGP 란

```
Border Gateway Protocol — internet 의 routing protocol.

역할:
  - AS (Autonomous System) 간 라우팅
  - "어느 IP 가 어느 path 로?"
  - peering / transit

scope:
  - 인터넷 전체의 routing 의사결정
  - AWS / GCP / Cloudflare 등 거대 ISP
  - 회사 단위 ASN 보유 (IANA 발급)
```

---

## 2. 왜 시니어 DevOps 가

```
대부분 회사 = 직접 BGP 운영 X.
하지만 시니어가 이해해야:
  
  - cloud LB / Direct Connect / VPN 의 underlying
  - multi-region failover (DNS vs BGP anycast)
  - DDoS 흡수 (Cloudflare anycast)
  - on-prem ↔ cloud 연결
  - Kubernetes MetalLB BGP mode
  - 큰 회사 = own ASN 운영
```

---

## 3. 핵심 개념

```
ASN (Autonomous System Number):
  - 고유 ID (예: AWS = AS16509, Cloudflare = AS13335)
  - public IP 의 owner

prefix advertisement:
  "이 IP block (1.2.3.0/24) 은 우리 AS"
  → BGP peer 에게 publish

peering:
  AS A ↔ AS B 직접 연결 (free / paid)

transit:
  AS A → AS B (transit provider) → 인터넷

BGP attribute:
  AS_PATH      : 거친 AS list (짧을수록 선호)
  LOCAL_PREF   : 같은 AS 안의 선호도
  MED          : 다른 AS 의 선호 (낮을수록 선호)
  COMMUNITY    : 임의 tag
```

---

## 4. anycast (★)

```
일반 unicast: 1 IP → 1 server
anycast: 1 IP → N server (BGP 가 가까운 곳 routing)

같은 1.1.1.1 (Cloudflare DNS):
  - 한국 user → 서울 server
  - 미국 user → 미국 server
  - 일본 user → 도쿄 server
  
→ 사용자가 가까운 server 자동 도달.
→ 한 region down = 다른 region 으로 자동 routing.
```

---

## 5. anycast 의 용도

```
1. DNS (Cloudflare 1.1.1.1, Google 8.8.8.8)
   - 모든 region 의 root server
   - global latency 최소

2. CDN edge POP
   - CloudFront / Cloudflare 의 200+ POP
   - 같은 IP, 다른 위치

3. DDoS scrubbing
   - 한 IP 에 traffic 가 N region 으로 분산
   - 한 region overload 도 다른 region 흡수

4. global load balancer
   - GCP Global LB (anycast IP)
   - AWS Global Accelerator
```

---

## 6. anycast vs DNS-based routing

| | anycast | DNS-based |
| --- | --- | --- |
| failover 시간 | < 1s (BGP) | DNS TTL (수분-수시간) |
| 정확도 | network topology | GeoIP database |
| TCP / UDP | 둘 다 | client 가 connect 후엔 X |
| 운영 부담 | 매우 큼 (ASN / BGP) | 단순 (Route53) |
| 비용 | 큼 | 저렴 |

→ **회사 대부분 = DNS-based**. 글로벌 SaaS / CDN = anycast.

---

## 7. BGP 위협

```
1. route hijack
   - 잘못된 / 악의적 prefix advertisement
   - 2008 Pakistan ↔ YouTube
   - 2018 Amazon Route 53 hijack
   
2. route leak
   - 잘못된 경로로 traffic 흐름
   - 회사 internal route 의 인터넷 노출

3. RPKI (Resource Public Key Infrastructure)
   - prefix 의 signed authorization
   - 인터넷 의 BGP 보안 표준
   - 2024 = 50%+ adoption
```

---

## 8. Kubernetes 와 BGP

```
MetalLB BGP mode:
  - cluster 의 LoadBalancer IP 를 BGP 로 router 에 advertise
  - on-prem k8s 의 "real" LB

Calico BGP:
  - pod CIDR 을 BGP 로 advertise
  - flat network (NAT 없이)

Cilium BGP control plane:
  - LB IP advertisement
  - service mesh + BGP
```

```yaml
# Cilium BGP example
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata: {name: prod-bgp}
spec:
  nodeSelector:
    matchLabels: {role: worker}
  virtualRouters:
    - localASN: 65001
      exportPodCIDR: true
      neighbors:
        - peerAddress: '10.0.0.1/32'
          peerASN: 65000
```

---

## 9. multi-region routing 결정

```
시나리오: 글로벌 service

option A: DNS-based (Route53 latency routing)
  + 단순
  + 비용 cheap
  - DNS cache delay
  - failover 느림

option B: GCP Global LB / AWS Global Accelerator (anycast)
  + 빠른 failover
  + 단일 IP
  + 좋은 latency
  - 비용
  - cloud lock-in

option C: 자체 anycast (own ASN)
  + control 완전
  + multi-cloud
  - 매우 비싸 / 복잡
  - 큰 회사만

대부분 회사 = A 또는 B.
```

---

## 10. AWS Global Accelerator

```
AWS 의 anycast LB:
  - 2 anycast IP
  - AWS backbone network (인터넷 우회)
  - region 별 health check
  - 자동 failover
  - TCP / UDP 둘 다 (NLB 와 통합)

비용:
  - $0.025/h per accelerator
  - + data transfer ($0.01-0.05/GB)

vs CloudFront:
  Global Accelerator: TCP/UDP, anycast IP
  CloudFront: HTTP/HTTPS, cache, edge POP
```

---

## 11. Cloudflare anycast (★)

```
모든 Cloudflare 의 200+ POP 에서:
  - 같은 IP advertise
  - 사용자의 가장 가까운 POP 로 자동
  - 한 POP down = 다른 POP 가 흡수
  - DDoS 흡수 (몇 Tbps 까지)

서비스:
  - DNS / CDN / WAF / DDoS
  - Workers (edge compute)
  - Spectrum (TCP/UDP)
  - Magic Transit (BGP-based DDoS)
```

→ "global 의 가장 단순한 접근" = Cloudflare.

---

## 12. on-prem ↔ cloud (BGP)

```
회사 datacenter ↔ AWS:
  - Direct Connect (BGP)
  - VPN 의 BGP (route exchange)
  
설정:
  - own ASN 또는 private ASN (64512-65535)
  - VLAN per VPC
  - BGP session per connection
  - prefix advertise / accept rule
```

---

## 13. 실전 troubleshoot

```bash
# AS path (외부 IP 까지)
mtr --report 8.8.8.8

# AS lookup
whois -h whois.radb.net AS16509

# 우리 prefix 의 visibility
# https://bgp.he.net/AS<id>
# https://radar.cloudflare.com/

# route advertisement
# bgpq3 / bgpq4 (filter list)

# 경로 추적
traceroute -A 8.8.8.8     # AS 표시
```

---

## 14. RPKI 검증

```bash
# 우리 prefix 의 ROA (Route Origin Authorization)
# https://rpki-validator.ripe.net/

# cloud provider 의 RPKI 의무
# Cloudflare / AWS / Telia / Hurricane Electric → RPKI valid only

# 회사 ASN 의 ROA 생성
# https://rpki.cloudflare.com/ (free)
```

---

## 15. CDN 의 path optimization

```
일반 internet routing:
  client → ISP A → ISP B → ISP C → origin
  
CDN POP (anycast):
  client → 가장 가까운 POP
  POP → CDN backbone (private network) → origin
  
→ 적게 hop / 안정 latency.
   "private backbone" 의 가치.
```

---

## 16. 함정

1. **anycast 의 misunderstanding** — TCP 의 session 은 한 server.
2. **ASN 없이 BGP** — 불가능. 등록 필요.
3. **route hijack 의심 무지** — RPKI / 모니터링.
4. **MED / LOCAL_PREF 잘못** — 비대칭 routing.
5. **multi-region 의 cost** — egress 큼.
6. **BGP convergence 느림** — 수십초 ~ 분.
7. **anycast 의 session affinity X** — 매 connection 다른 server.

---

## 17. 관련

- [[networking-ops|↑ networking-ops]]
- [[dns]]
- [[load-balancer-types]]
- [[cdn]]
- [[../cloud-aws/cloud-aws|↗ AWS Direct Connect]]
