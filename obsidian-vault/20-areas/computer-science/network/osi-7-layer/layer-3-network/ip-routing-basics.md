---
title: "IP 라우팅 기초 — Routing Table, Longest Prefix Match"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:45:00+09:00
tags:
  - network
  - layer-3
  - routing
  - ip
---

# IP 라우팅 기초 — Routing Table, Longest Prefix Match

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 라우팅 테이블 / Longest Prefix / Default GW / FIB / RIB |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

> 라우팅 프로토콜 (RIP / OSPF / BGP) 깊이는 [[../../routing/routing|↗ 라우팅 폴더]].

---

## 1. 한 줄 정의

**IP 패킷을 목적지까지 next-hop 라우터에 전달** 하는 결정 — 라우팅 테이블 검색과
Longest Prefix Match.

---

## 2. 라우팅 vs Forwarding

| 측면 | Routing | Forwarding |
| --- | --- | --- |
| 무엇 | 경로 학습 / 결정 (Control Plane) | 패킷을 next hop 으로 (Data Plane) |
| 빈도 | 가끔 (이벤트 / 주기) | 매 패킷 |
| 알고리즘 | Dijkstra, Bellman-Ford, BGP path selection | Longest Prefix Match |
| 구현 | SW (라우팅 데몬) | HW (ASIC, TCAM) |
| 데이터 | **RIB** (Routing Information Base) | **FIB** (Forwarding Information Base) |

---

## 3. 라우팅 테이블 구조

```
+-------------------+----------+----------+---------+-------+
| Destination       | Mask     | Next Hop | Iface   | Metric|
+-------------------+----------+----------+---------+-------+
| 0.0.0.0           | /0       | 192.168.1.1 | eth0  | 1     |
| 192.168.1.0       | /24      | -        | eth0    | 0     |
| 10.0.0.0          | /8       | 10.0.0.1 | eth1    | 1     |
| 8.8.8.8           | /32      | 192.168.1.1 | eth0  | 5     |
| ...               | ...      | ...      | ...     | ...   |
+-------------------+----------+----------+---------+-------+
```

### 3.1 필드
- **Destination + Mask** — 목적 네트워크 (CIDR)
- **Next Hop** — 다음 라우터의 IP
- **Interface** — 어느 NIC 로 보낼지
- **Metric** — 비용 (낮을수록 우선)
- **Admin Distance** — 라우팅 소스 신뢰도

### 3.2 OS 별 확인

```bash
# Linux 모던
ip route
ip route show table all

# Linux 옛
route -n

# macOS
netstat -rn
route get 8.8.8.8

# Windows
route print
```

---

## 4. Longest Prefix Match

여러 매칭 중 **가장 긴 마스크** 선택:

```
Routing Table:
- 0.0.0.0/0     → GW1 (default)
- 10.0.0.0/8    → GW2
- 10.10.0.0/16  → GW3
- 10.10.10.0/24 → GW4

목적지 10.10.10.5 → 모두 매칭 → /24 가 가장 김 → GW4 선택
목적지 10.10.20.5 → /16 까지 매칭 → GW3
목적지 10.20.30.5 → /8 까지 매칭 → GW2
목적지 192.168.1.1 → /0 만 → GW1
```

### 4.1 구현 자료구조

- **Trie** — 비트 단위 트리 (가장 단순)
- **Radix Tree** (PATRICIA) — 압축 trie, Linux 커널
- **HW** — TCAM (Ternary CAM) — 1 사이클 검색

자세히 → [[../../../data-structure/tries/tries|↗ Tries]]

---

## 5. Default Gateway

```
0.0.0.0/0 → next-hop = 192.168.1.1
```

모든 다른 라우트가 매치 안 될 때 사용. 가정 / 작은 LAN 에선 라우터 자체.

### 5.1 다중 default gateway
- WAN 이중화 — 활성/대기 또는 ECMP
- VRRP / HSRP 로 가상 GW

---

## 6. 정적 vs 동적 라우팅

### 6.1 정적 (Static)

```bash
# Linux
sudo ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0

# Cisco
ip route 10.0.0.0 255.0.0.0 192.168.1.1
```

- 단순, 변경 적은 망
- 작은 사무실, edge 라우터

### 6.2 동적 (Dynamic)

- 라우팅 프로토콜 (RIP, OSPF, BGP) 가 자동 학습
- 대규모 망, 변경 잦음

자세히 → [[../../routing/routing|↗ 라우팅 폴더]]

---

## 7. Administrative Distance (Cisco)

여러 소스에서 같은 destination 학습 시 우선순위:

| 소스 | AD |
| --- | --- |
| Connected | 0 |
| Static | 1 |
| **eBGP** | 20 |
| **OSPF** | 110 |
| **RIP** | 120 |
| **iBGP** | 200 |
| Unknown | 255 (사용 X) |

낮을수록 신뢰. 같은 AD 면 metric 비교.

---

## 8. ECMP (Equal-Cost Multi-Path)

같은 metric 의 여러 경로 — 부하 분산:

```
A 가 B 에 → 두 경로 (10.0.0.1, 10.0.0.2) 모두 metric 100
→ 패킷 별 분산 (5-tuple hash)
```

- 한 흐름은 같은 경로 (순서 보장)
- L3/L4 hash — Src/Dst IP, Src/Dst Port, Protocol

---

## 9. CIDR — Classless Inter-Domain Routing

자세히 → [[../../ip/cidr-subnetting]]

### 9.1 표기
```
192.168.1.0/24     # 마스크 24 bit
2001:db8::/32      # IPv6
```

### 9.2 Supernet (요약)
```
192.168.0.0/24, 192.168.1.0/24, 192.168.2.0/24, 192.168.3.0/24
  → 192.168.0.0/22 (4 개를 1 개로)
```

BGP 라우팅 테이블 크기 줄임 → "Route Aggregation".

---

## 10. 라우팅 결정 과정 (호스트 / 라우터)

```
1. 목적지 IP 추출
2. Loopback (127.x) → loopback 인터페이스
3. 자신의 IP → loopback
4. 같은 서브넷 (자기 IP & 마스크)? → ARP 후 직접 전달
5. 라우팅 테이블 검색 (Longest Prefix Match)
6. Next hop 의 MAC 을 ARP 로 찾음
7. L2 프레임 송신
```

---

## 11. 라우터 안의 흐름

```
패킷 수신
  ↓
입력 인터페이스 ACL
  ↓
TTL -= 1, TTL=0 면 ICMP TTL Exceeded 후 폐기
  ↓
헤더 checksum 검증 / 재계산 (IPv4 만)
  ↓
FIB 검색 (Longest Prefix Match) — HW (TCAM)
  ↓
Next hop ARP / NDP — 캐시 있으면 그대로
  ↓
출력 인터페이스 ACL / QoS / Rate limit
  ↓
새 L2 프레임 생성 (Src/Dst MAC 갱신)
  ↓
출력
```

---

## 12. 라우팅 함정

### 함정 1 — 라우팅 루프
A → B → A → B → ... TTL 0 까지 무한. 라우팅 프로토콜이 방지해야.

### 함정 2 — 비대칭 경로
- A → B 와 B → A 가 다른 경로
- Stateful 방화벽 / NAT 문제
- BGP / ECMP 환경에서 흔함

### 함정 3 — Black hole
잘못된 라우트 → next hop 이 폐기. ping 으로 진단.

### 함정 4 — Default route 누락
LAN 만 통신, 외부 X. 호스트 / 라우터 모두 확인.

### 함정 5 — MTU mismatch
경로 중 작은 MTU + ICMP 차단 → PMTUD 실패. [[fragmentation-mtu]] 참조.

### 함정 6 — IP spoofing
출발 IP 위조. uRPF (unicast Reverse Path Forwarding) 로 방어.

### 함정 7 — Source-Based Routing
일반 라우팅은 dst 기반. Policy-Based Routing (PBR) 으로 src 기반 가능.

---

## 13. 라우팅 디버깅 명령

### Linux
```bash
ip route get 8.8.8.8                # 어떤 라우트로?
ip route show 10.0.0.0/8
ip rule list                         # PBR rules
traceroute -I 8.8.8.8
mtr -t 8.8.8.8

# 패킷 흐름 추적
sudo nft monitor trace
sudo iptables -t raw -A PREROUTING -p icmp -j TRACE
```

### Cisco
```
show ip route
show ip route 8.8.8.8
show ip cef           # FIB (HW)
show ip arp
debug ip routing
traceroute 8.8.8.8
```

---

## 14. 학습 자료

- RFC 1812 (Requirements for IP Version 4 Routers)
- **TCP/IP Illustrated Vol.1** Ch. 9
- **Routing TCP/IP Vol.1 & 2** (Doyle / Carroll) — Cisco 책
- Cisco "Configuring IP Routing"

---

## 15. 관련

- [[arp]] — Next hop MAC 해결
- [[fragmentation-mtu]] — MTU 와 라우팅
- [[../../routing/routing|↗ 라우팅 프로토콜 폴더]] (RIP/OSPF/BGP)
- [[../../ip/cidr-subnetting]] — Longest Prefix 의 기반
- [[layer-3-network]] — 상위
