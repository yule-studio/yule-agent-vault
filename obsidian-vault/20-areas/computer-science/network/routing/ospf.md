---
title: "OSPF — Open Shortest Path First"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:35:00+09:00
tags:
  - network
  - routing
  - ospf
  - link-state
  - dijkstra
---

# OSPF — Open Shortest Path First

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | LSA / Area / DR/BDR / SPF / Cost / OSPFv3 |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **분류** | Link State / IGP |
| **알고리즘** | Dijkstra SPF |
| **메트릭** | Cost (10^8 / Bandwidth) |
| **광고 방식** | LSA flooding |
| **Convergence** | 초 단위 |
| **프로토콜 번호** | IP Protocol 89 |
| **AD (Cisco)** | 110 |
| **Multicast** | 224.0.0.5 (All OSPF), 224.0.0.6 (DR) |
| **RFC** | 2328 (v2), 5340 (v3) |

---

## 1. 한 줄 정의

**모든 라우터가 망의 전체 토폴로지를 알고** Dijkstra SPF 로 최단 경로 계산.
RFC 2328 (1998) — 인터넷의 IGP 표준.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1989 | OSPFv1 (RFC 1131) |
| 1991 | **OSPFv2 (RFC 1247 → 2328)** — 표준 |
| 1999 | **OSPFv3 (RFC 2740 → 5340)** — IPv6 |
| 2003 | TE 확장 |

---

## 3. Link State 원리

```
각 라우터:
1. Hello 로 이웃 발견
2. 자기 link 상태 (LSA) broadcast (flooding)
3. 모든 LSA 모아 LSDB (Link State Database) 구축
4. LSDB 에서 자기를 root 로 SPF (Dijkstra) → 최단 경로
5. 결과를 RIB / FIB 에 입력
```

→ 모든 라우터의 LSDB **동일** = SPF 결과 일관성.

---

## 4. LSA — Link State Advertisement

### 4.1 종류 (OSPFv2)

| Type | 이름 | 광고 범위 |
| --- | --- | --- |
| 1 | **Router LSA** | Area 내 |
| 2 | **Network LSA** | Area 내 (DR 가 생성) |
| 3 | **Summary LSA** (네트워크) | ABR 가 다른 Area 로 |
| 4 | **Summary LSA** (ASBR) | ABR 가 다른 Area 로 |
| 5 | **External LSA** | ASBR 가 (전체) |
| 7 | **NSSA External** | NSSA Area 만 |
| 9-11 | **Opaque** | 확장 (MPLS TE) |

### 4.2 LSA 흐름
```
Router → Hello → 이웃
Router → DD (Database Description) → 이웃
이웃 → LSR (Link State Request) → Router
Router → LSU (Link State Update) → 이웃
이웃 → LSAck → Router
```

---

## 5. Area — 망 분할

### 5.1 왜
- 한 Area 안 LSDB 동일 — 큰 망에선 비용 ↑
- → Area 로 나눠 SPF 계산 범위 제한

### 5.2 종류

```
Backbone (Area 0)
   ↓
ABR (Area Border Router)
   ↓
Area 1, 2, 3...

모든 non-backbone Area 는 Area 0 에 연결 필수
```

### 5.3 Area 타입

| 종류 | 특징 |
| --- | --- |
| **Backbone (0)** | 모든 Area 의 중심 |
| **Standard** | 모든 LSA 허용 |
| **Stub** | External LSA (Type 5) 차단 |
| **Totally Stub** | Type 3,4,5 차단 (Cisco 만) |
| **NSSA** (Not-So-Stubby) | Type 7 (NSSA External) 허용 |
| **Totally NSSA** | 더 제한 |

### 5.4 라우터 역할

- **Internal Router** — 한 Area 안
- **ABR** (Area Border Router) — 두 Area 걸침
- **ASBR** (AS Boundary Router) — 다른 AS 와 redistribution
- **Backbone Router** — Area 0

---

## 6. Cost — 메트릭

```
Cost = 10^8 / Bandwidth (bps)

10 Gbps  → cost 0.01 (→ 1 로)
1 Gbps   → cost 0.1
100 Mbps → cost 1
10 Mbps  → cost 10
1.5 Mbps → cost 65

경로 cost = 모든 link cost 합
```

수동 설정 가능: `ip ospf cost 10`.

→ **빠른 link 우선** 자동 선택.

---

## 7. DR / BDR — Designated Router

### 7.1 왜
- Broadcast 망 (Ethernet) 에서 N 라우터가 N(N-1)/2 adjacency → 폭발
- → 한 DR 가 중심, 모두 DR 와만 adjacency

### 7.2 선출 (Election)

```
1. Priority 가장 높은 라우터 (기본 1, 0 이면 X)
2. Tie → Router ID 가 큰 쪽
```

### 7.3 DR / BDR
- DR — 활성
- BDR — DR 죽으면 즉시 승계 (선출 비용 회피)
- 다른 라우터 = DROther

### 7.4 통신
- 모두 → DR (224.0.0.6, AllDRouters)
- DR → 모두 (224.0.0.5, AllSPFRouters)

---

## 8. Hello Protocol

```
Hello packet 매 10 초 (LAN), 30 초 (NBMA)
Dead Interval = 4 × Hello (40 초)

Hello 안 옴 → 이웃 죽음 → SPF 재계산
```

### Hello 내용
- Router ID
- Hello / Dead interval
- Area ID
- Authentication
- Neighbor list

---

## 9. SPF — Dijkstra

각 라우터가 자기를 root 로:

```python
def dijkstra(graph, source):
    dist = {v: inf for v in graph}
    dist[source] = 0
    pq = [(0, source)]
    while pq:
        d, u = heappop(pq)
        if d > dist[u]: continue
        for v, weight in graph[u]:
            new_d = d + weight
            if new_d < dist[v]:
                dist[v] = new_d
                heappush(pq, (new_d, v))
    return dist
```

자세히 → [[../../algorithm/shortest-path/shortest-path]]

### 9.1 Incremental SPF (iSPF)
- 변화된 부분만 다시 — 전체 SPF 회피
- 모던 라우터의 최적화

---

## 10. OSPFv3 (IPv6, RFC 5340)

### 10.1 차이점
- IPv6 주소
- Link-local 주소로 OSPF 운영
- LSA 형식 변경 (Prefix LSA, Link LSA)
- 인증은 IPsec (자체 X)
- Multi-instance (한 link 에 여러 OSPF 인스턴스)

### 10.2 OSPFv2 와 OSPFv3 공존
- 한 라우터에 둘 다 (각각 IPv4/IPv6)
- Address-Family 지원 (MP-OSPFv3) — 한 인스턴스가 양쪽

---

## 11. 인증

| 방법 | 강도 |
| --- | --- |
| **None** | 0 |
| **Simple (Plain)** | 약 (스니핑) |
| **MD5** | 보통 (sequence 보호) |
| **SHA-256+** | 모던 (RFC 5709) |

```
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 SECRET
```

---

## 12. Cisco 설정 예

```
router ospf 1
  router-id 1.1.1.1
  area 0 authentication message-digest
  network 192.168.1.0 0.0.0.255 area 0
  network 10.0.0.0 0.255.255.255 area 1
  passive-interface default
  no passive-interface Gi0/0
  default-information originate    # default route 광고

interface Gi0/0
  ip ospf 1 area 0
  ip ospf cost 10
  ip ospf hello-interval 5
  ip ospf dead-interval 20
  ip ospf message-digest-key 1 md5 SECRET
```

### Linux FRR / Quagga

```
router ospf
  ospf router-id 1.1.1.1
  network 192.168.1.0/24 area 0.0.0.0
  passive-interface default
  no passive-interface eth0
```

---

## 13. 디버깅

```
show ip ospf neighbor              # 이웃 상태
show ip ospf interface             # 인터페이스
show ip ospf database              # LSDB
show ip route ospf                 # OSPF 경로
debug ip ospf events
debug ip ospf packet
```

### 상태
- **Down** — 통신 없음
- **Init** — Hello 받음
- **2-Way** — 양방향 hello
- **ExStart** — 협상 시작
- **Exchange** — DD 교환
- **Loading** — LSR/LSU
- **Full** — adjacency 완료 ✅

---

## 14. 함정

### 함정 1 — Router ID 변경 시 OSPF 재시작
모든 OSPF 인스턴스 reset → 망 전체 영향.

### 함정 2 — Area 0 단절
백본 끊김 → 라우팅 분할. Virtual Link 로 임시 복구.

### 함정 3 — DR 의 단일 장애점
BDR 가 즉시 승계하지만 잠시 미수렴.

### 함정 4 — MTU mismatch
이웃 간 MTU 다르면 ExStart 에서 stuck.

### 함정 5 — Hello / Dead 불일치
다르면 이웃 X.

### 함정 6 — Auto-summary
CIDR 환경에서 의도치 않은 경로.

### 함정 7 — LSA flooding 폭증
Flapping link → 망 전체 부담. Dampening.

### 함정 8 — passive interface 누락
WAN / public 으로 OSPF 광고 → 보안 사고.

---

## 15. IS-IS — OSPF 의 사촌

ISO 표준 (1992) — 비슷한 LS 프로토콜:
- ISP 백본에서 흔함 (OSPF 대신)
- TLV 기반 — 확장성 ↑
- L1 / L2 hierarchy (OSPF 의 Area 와 유사)
- IPv6 통합 단일 인스턴스
- 더 stable, 덜 vendor lock-in

---

## 16. 학습 자료

- RFC 2328 (OSPFv2), RFC 5340 (OSPFv3), RFC 4222 (Performance Implications)
- **OSPF Network Design Solutions** (Thomas)
- **OSPF and IS-IS** (Doyle)
- Cisco "OSPF Configuration Guide"

---

## 17. 관련

- [[bgp]] — EGP
- [[../../algorithm/shortest-path/shortest-path]] — Dijkstra
- [[routing]] — 라우팅 hub
- [[rip]] — DV 비교
