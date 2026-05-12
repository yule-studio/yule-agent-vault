---
title: "EIGRP — Enhanced Interior Gateway Routing Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:45:00+09:00
tags:
  - network
  - routing
  - eigrp
  - cisco
---

# EIGRP — Enhanced Interior Gateway Routing Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DUAL / 복합 메트릭 / Successor / FS |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **분류** | Advanced Distance Vector (Hybrid) / IGP |
| **알고리즘** | DUAL (Diffusing Update Algorithm) |
| **메트릭** | Composite (Bandwidth + Delay + Reliability + Load + MTU) |
| **프로토콜 번호** | IP Protocol 88 |
| **AD (Cisco)** | 90 (internal) / 170 (external) |
| **Multicast** | 224.0.0.10 |
| **RFC** | 7868 (informational, 2016) |
| **출처** | Cisco proprietary → 2013 IETF |

---

## 1. 한 줄 정의

Cisco 의 **Hybrid 라우팅** — Distance Vector 의 단순함 + Link State 의 빠른
수렴 + 복합 메트릭. 2013 RFC 7868 로 공개.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1985 | IGRP — Cisco proprietary DV |
| 1993 | **EIGRP** — IGRP 의 발전 |
| 2013 | RFC 7868 (informational) — 공개 |
| 현재 | 일부 Cisco 환경 사용. 모던 망은 OSPF / BGP. |

---

## 3. EIGRP 의 5 가지 핵심

### 3.1 DUAL — Diffusing Update Algorithm
- Distance Vector 이지만 loop-free 보장
- 변화 시 빠른 수렴
- SRTT 와 Hello 로 이웃 상태

### 3.2 5 가지 메트릭 (K 값)

```
Metric = ((K1 × BW + K2 × BW / (256 - load) + K3 × Delay) × (K5 / (Reliability + K4))) × 256

기본 K1=K3=1, K2=K4=K5=0:
Metric = (BW + Delay) × 256

BW = 10^7 / min(bandwidth) (kbps)
Delay = sum of delays / 10 (μs)
```

→ 빠른 link + 짧은 delay 우선.

### 3.3 Successor / Feasible Successor

- **Successor** — 최적 경로 (라우팅 테이블)
- **Feasible Successor** — backup 경로 (Reported Distance < Feasible Distance 의 조건 만족)
- Successor 죽으면 즉시 FS 로 — **수 ms 수렴**

### 3.4 Hello / RTP

- Hello 매 5 초 (LAN) / 60 초 (WAN)
- Hold time 3 × Hello
- RTP (Reliable Transport Protocol) — 일부 메시지 ACK

### 3.5 Partial / Bounded Update

- 변화만 광고 (전체 테이블 X — RIP 와 다름)
- 영향받는 라우터만 (전체 broadcast X)

---

## 4. 메트릭 계산 예

```
S1 ─ 100M, 100μs ─ R1 ─ 1G, 10μs ─ R2 ─ 100M, 100μs ─ S2

S1→S2 cost:
  min BW = 100M = 100,000 kbps
  BW = 10^7 / 100,000 = 100
  Delay = (100 + 10 + 100) / 10 = 21
  Metric = (100 + 21) × 256 = 30,976
```

→ 가장 좁은 link 가 결정 + 누적 delay.

---

## 5. Successor / Feasible Successor 자세히

```
R1 의 후보 경로:
  Path A → R3: FD=100  (Feasible Distance = 자기까지)
  Path B → R4: FD=150
  Path C → R5: FD=200

Successor = Path A (FD 가장 작음)

Feasible Successor 조건:
  Path B 의 Reported Distance (RD) < Successor 의 FD (100)
  Path C 의 Reported Distance (RD) < Successor 의 FD (100)
  
→ 만족하면 FS, 안 만족하면 보유 X (DUAL 의 query 필요)
```

### Active vs Passive
- **Passive** — 안정 (Successor 가짐)
- **Active** — 변화 발생 → 이웃들에 query, 응답 대기
- Active 너무 길면 SIA (Stuck In Active) — 경로 무효화

---

## 6. EIGRP 패킷 종류

| 종류 | 용도 |
| --- | --- |
| **Hello** | 이웃 발견 / 유지 |
| **Update** | 라우팅 정보 (RTP) |
| **Query** | 변화 시 이웃에 정보 요청 |
| **Reply** | Query 응답 |
| **Acknowledgment** | RTP 의 ACK |

---

## 7. EIGRP for IPv6 / Named Mode

### 7.1 IPv6 EIGRP
- IPv4 EIGRP 와 비슷, IPv6 주소

### 7.2 Named EIGRP
- 옛 "classic" 모드의 단점 해결
- 모든 address-family 한 인스턴스
- 모던 권장

---

## 8. Cisco 설정 예

```
router eigrp 100
  network 192.168.1.0 0.0.0.255
  no auto-summary
  passive-interface default
  no passive-interface Gi0/0
  
  ! Named mode
router eigrp MY-EIGRP
  address-family ipv4 unicast autonomous-system 100
    network 192.168.1.0/24
    af-interface Gi0/0
      no passive-interface
```

---

## 9. EIGRP vs OSPF

| 측면 | EIGRP | OSPF |
| --- | --- | --- |
| 표준 | Cisco (RFC 7868 informational) | 진정한 표준 (RFC 2328) |
| Vendor | Cisco 중심 | 모든 vendor |
| 알고리즘 | DUAL | Dijkstra SPF |
| 수렴 | 매우 빠름 (FS) | 빠름 |
| 메트릭 | 복합 | Cost (BW) |
| CPU | 낮음 | 보통 |
| Area | 평평 | Hierarchical |
| Multipath | 동일 cost + variance (unequal) | 동일 cost ECMP |

→ **OSPF 가 표준**. EIGRP 는 Cisco 환경.

---

## 10. 함정

### 함정 1 — K 값 변경
양쪽 K 값 같아야 이웃 형성. 변경 시 모두 변경.

### 함정 2 — SIA (Stuck In Active)
큰 망에서 query 가 멀리 전파 → 응답 늦음 → SIA. **Stub** 라우터 (`eigrp stub`).

### 함정 3 — Variance
unequal cost multipath — Variance N 면 metric × N 이내 경로 활용. 디버깅 어려움.

### 함정 4 — Auto-summary
CIDR 환경에서 의도 X 경로.

### 함정 5 — Cisco lock-in
다른 vendor 와 호환성 (RFC informational 후 일부 vendor 지원하지만 제한적).

### 함정 6 — Stub 누락
Hub-Spoke 에서 spoke 가 transit 사용 시 비효율.

---

## 11. 학습 자료

- RFC 7868 (EIGRP)
- **Routing TCP/IP Vol.1** (Doyle)
- Cisco "EIGRP Configuration Guide"

---

## 12. 관련

- [[ospf]] — IGP 대표
- [[rip]] — DV 의 원시
- [[bgp]] — EGP
- [[routing]] — 라우팅 hub
