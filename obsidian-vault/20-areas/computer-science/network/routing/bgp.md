---
title: "BGP — Border Gateway Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:40:00+09:00
tags:
  - network
  - routing
  - bgp
  - asn
---

# BGP — Border Gateway Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Path Vector / Attributes / iBGP/eBGP / RR / Hijack / RPKI |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **분류** | Path Vector / EGP |
| **메트릭** | Path attributes (복잡) |
| **전송** | TCP 179 |
| **Convergence** | 분 단위 (큰 망) |
| **AD (Cisco)** | 20 (eBGP) / 200 (iBGP) |
| **RFC** | 4271 (BGP-4), 4760 (MP-BGP), 6480 (RPKI) |
| **트래픽** | 인터넷의 모든 AS 간 라우팅 |

---

## 1. 한 줄 정의

**AS (Autonomous System) 간 라우팅** 의 사실상 유일한 프로토콜. RFC 4271 — 인터넷의
"glue". Path Vector — 경로 + policy 기반.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1989 | BGP-1 (RFC 1105) |
| 1990 | BGP-2 |
| 1995 | BGP-3 |
| 1995 | **BGP-4 (RFC 1771 → 4271)** — CIDR 지원, 표준 |
| 2007 | MP-BGP (RFC 4760) — IPv6, MPLS L3VPN |
| 2012 | 4-byte AS (RFC 6793) |
| 2014 | **RPKI** (RFC 6810) — 보안 |

---

## 3. AS — Autonomous System

```
AS = 한 관리 주체 (ISP, 회사) 하의 IP 네트워크 그룹
ASN (Autonomous System Number) — 16 bit (1-65535) 또는 32 bit (4-byte AS)
```

### 3.1 ASN 종류
- **Public** — IANA 등록, 인터넷 광고
- **Private** — 64512-65534 (16 bit), 4200000000-4294967294 (32 bit)
- **Reserved** — 0, 23456 (AS_TRANS), 65535 등

### 3.2 주요 ASN

| ASN | AS |
| --- | --- |
| 13335 | Cloudflare |
| 15169 | Google |
| 16509 | AWS |
| 32934 | Facebook |
| 8075 | Microsoft |
| 9009 | M247 |
| 4766 | Korea Telecom |
| 4775 | Globe Telecom |

확인: https://bgp.he.net, `whois -h whois.cymru.com " -v AS15169"`.

---

## 4. iBGP vs eBGP

### 4.1 eBGP — External BGP
- 다른 AS 와
- TTL = 1 (직접 인접)
- 광고 받은 prefix 의 next-hop 변경
- AS Path 에 자기 ASN 추가

### 4.2 iBGP — Internal BGP
- 같은 AS 내부
- TTL = 255
- next-hop 변경 X
- AS Path 변경 X
- **Full Mesh** 필요 (모든 iBGP peer 가 직접 연결)

→ 큰 AS 에서 full mesh = N(N-1)/2 → **Route Reflector** 또는 **Confederation** 사용.

---

## 5. Path Attributes

BGP UPDATE 의 핵심 — 경로 선택의 기반.

### 5.1 Well-Known Mandatory
모든 BGP peer 가 인식 + 모든 UPDATE 에 포함.

| Attribute | 의미 |
| --- | --- |
| **ORIGIN** | IGP (0) / EGP (1) / Incomplete (2) |
| **AS_PATH** | 거쳐 온 AS 목록 (loop 방지 + 길이 = metric) |
| **NEXT_HOP** | 다음 hop IP |

### 5.2 Well-Known Discretionary
인식하지만 선택.

- **LOCAL_PREF** — 자기 AS 안 우선순위 (높을수록 우선)
- **ATOMIC_AGGREGATE** — 집계 정보

### 5.3 Optional Transitive
모르면 무시 (전달).

- **AGGREGATOR** — 집계자 정보
- **COMMUNITY** — 정책 태그
- **EXTENDED COMMUNITY** — VPN, L2VPN

### 5.4 Optional Non-Transitive
모르면 폐기.

- **MED** (Multi-Exit Discriminator) — AS 진입점 우선 (낮을수록 우선)
- **CLUSTER_LIST** — Route Reflector
- **ORIGINATOR_ID** — Route Reflector

---

## 6. BGP 경로 선택 (Best Path)

여러 경로 중 어느 하나 선택 — Cisco 순서:

```
1. WEIGHT (Cisco proprietary, 높을수록 우선) [기본 32768]
2. LOCAL_PREF (높을수록 우선) [기본 100]
3. Locally originated (자기가 만든 거)
4. AS_PATH 짧을수록 우선
5. ORIGIN (IGP < EGP < Incomplete)
6. MED (낮을수록 우선)
7. eBGP > iBGP (eBGP 우선)
8. IGP cost to next-hop 짧을수록
9. Oldest path (eBGP 만)
10. Router ID 낮을수록
11. Cluster List 짧을수록
12. Peer IP 낮을수록
```

→ "Pick"은 보통 1-4 에서 결정. 외워두기.

---

## 7. UPDATE 메시지

```
[Withdrawn Routes — 더 이상 광고 X]
[Path Attributes]
[NLRI — Network Layer Reachability Info (prefix 목록)]
```

### KEEPALIVE
- 60 초마다 (기본)
- Hold Time (180 초) 안 받으면 peer dead

### NOTIFICATION
- 오류 알림 후 연결 종료

---

## 8. BGP State Machine

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
                                                         ↓
                                                   UPDATE 교환
```

### 상태
- **Idle** — 시작 / 종료
- **Connect / Active** — TCP 연결 시도
- **OpenSent** — OPEN 보냄
- **OpenConfirm** — OPEN 받음, KEEPALIVE 대기
- **Established** — UPDATE 교환 가능 ✅

---

## 9. Route Reflector (RFC 4456)

iBGP full mesh 문제 해결:

```
N 라우터 full mesh = N(N-1)/2 세션 (100 → 4950)

Route Reflector:
- 1 RR + N 클라이언트 = N 세션
- RR 가 한 클라가 받은 경로를 다른 클라에 전달 (iBGP 의 split horizon 우회)
```

### Cluster ID
- RR + 클라이언트 = 1 cluster
- Originator ID + Cluster List 로 loop 방지

### Confederation
- AS 를 sub-AS 들로 분할
- 각 sub-AS 안에선 full mesh
- 외부에는 단일 AS 로 보임

---

## 10. BGP Security — 핵심 이슈

### 10.1 BGP Hijacking

가짜 prefix 광고로 트래픽 가로채기:

#### 사례
- **2008 Pakistan ↔ YouTube** — PT 가 자기 prefix 로 YouTube 광고, 전 세계 끊김
- **2018 Amazon Route 53** — eNet 이 가짜 광고, 암호화폐 도난
- **2021 Facebook** — 자기 BGP 광고 철회로 자기 자신 끊김 (DNS 도)

### 10.2 Route Leak
- 정책 위반 광고 (Customer routes 를 다른 customer 에 전달)
- 트래픽 폭주 / 우회

### 10.3 RPKI — Resource Public Key Infrastructure (RFC 6480)

```
1. IANA / RIR 가 ASN, prefix 의 소유자 인증
2. ROA (Route Origin Authorization) — "AS X 가 prefix Y 를 광고할 권한"
3. BGP 라우터가 ROA 검증 → invalid 면 폐기 또는 낮은 우선순위
```

### 10.4 BGPsec (RFC 8205)
- 경로 자체 서명 — 모든 AS 가 서명
- 채택률 낮음 (CPU 부담, 점진)

### 10.5 보안 도구
- **BGP Monitor** — BGPMon, RIPE Atlas
- **MANRS** (Mutually Agreed Norms for Routing Security)

---

## 11. Cisco 설정 예

```
router bgp 65000
  bgp router-id 1.1.1.1
  neighbor 192.168.1.2 remote-as 65001     # eBGP
  neighbor 10.0.0.2 remote-as 65000        # iBGP
  network 192.168.1.0/24                    # 광고할 prefix
  
  ! Route Reflector
  neighbor 10.0.0.3 route-reflector-client
  
  ! 정책
  neighbor 192.168.1.2 route-map IN in
  neighbor 192.168.1.2 prefix-list LIST in
  
  ! 인증
  neighbor 192.168.1.2 password SECRET
```

### Linux FRR / BIRD / GoBGP

```
# FRR
router bgp 65000
  bgp router-id 1.1.1.1
  neighbor 192.168.1.2 remote-as 65001
  address-family ipv4 unicast
    network 192.168.1.0/24
  exit-address-family
```

---

## 12. BGP 디버깅

```
show ip bgp summary                    # peer 상태
show ip bgp neighbors                  # 상세
show ip bgp                            # 전체 경로
show ip bgp 192.168.1.0/24             # 특정 prefix
show ip route bgp                      # BGP 가 선택한 경로

debug bgp updates
debug bgp events
```

### 외부에서 보기

```
# RIPE Atlas, bgp.he.net 등 looking glass
$ whois -h whois.cymru.com " -v 8.8.8.8"
$ traceroute --as-path-lookups 8.8.8.8
```

---

## 13. MP-BGP — Multiprotocol BGP

### 13.1 동기
- IPv4 외 (IPv6, MPLS, L2VPN) 도 광고

### 13.2 AFI / SAFI

```
AFI (Address Family Identifier):
  1 = IPv4, 2 = IPv6, 25 = L2VPN

SAFI (Subsequent AFI):
  1 = Unicast, 2 = Multicast, 4 = MPLS, 128 = VPNv4
```

→ MPLS L3VPN, EVPN 의 기반.

---

## 14. BGP 활용

### 14.1 ISP
- 인터넷 라우팅
- Multi-homing (여러 ISP 연결)

### 14.2 데이터센터
- iBGP / eBGP 로 CLOS Fabric
- Free range routing

### 14.3 SDN
- BGP-LS (Link State 통신)
- SRv6 control plane

### 14.4 Anycast
- 같은 prefix 를 여러 곳에서 광고
- DNS / CDN 의 기반 (Cloudflare 1.1.1.1)

---

## 15. 함정

### 함정 1 — BGP 의 느린 수렴
큰 변화 시 분 단위. BGP-FRR (Fast Reroute) 로 보완.

### 함정 2 — Full mesh iBGP
N(N-1)/2 → RR 또는 Confederation.

### 함정 3 — Hijacking 의 위험
RPKI ROA 등록 필수.

### 함정 4 — IGP 와 iBGP 의 next-hop
재광고 시 next-hop 설정 ("next-hop-self").

### 함정 5 — 정책 검증 없음
임의 광고 받음 → Prefix List / AS Path filter 필수.

### 함정 6 — Memory 사용
글로벌 라우팅 테이블 ~ 1M routes, 수 GB RAM 필요.

### 함정 7 — Convergence storm
링크 flap → 라우팅 폭증. Dampening.

---

## 16. 학습 자료

- RFC 4271 (BGP-4), RFC 4456 (RR), RFC 6480 (RPKI)
- **BGP** (Iljitsch van Beijnum)
- **Internet Routing Architectures** (Halabi)
- **Routing TCP/IP Vol.2** (Doyle)
- NANOG / RIPE / APNIC 자료
- Cloudflare blog — BGP 시리즈

---

## 17. 관련

- [[ospf]] — IGP
- [[mpls]] — MP-BGP 위 L3VPN
- [[routing]] — 라우팅 hub
- [[../topics/topics]] — BGP 사고 (Facebook 2021 등)
