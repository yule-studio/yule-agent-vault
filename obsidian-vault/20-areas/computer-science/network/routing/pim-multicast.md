---
title: "PIM — Protocol Independent Multicast"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:55:00+09:00
tags:
  - network
  - routing
  - multicast
  - pim
---

# PIM — Protocol Independent Multicast

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | PIM-DM / PIM-SM / SSM / Bidir / RPF |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

> Multicast 자체 (IGMP / Multicast 주소 / IGMP Snooping) 는
> [[../osi-7-layer/layer-3-network/multicast-igmp|↗ Multicast & IGMP]]. 이 노트는
> **라우터 간 multicast 전달** 깊이.

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **종류** | DM (Dense), SM (Sparse), SSM (Source-Specific), Bidir |
| **프로토콜 번호** | IP Protocol 103 |
| **Multicast 주소** | 224.0.0.13 (PIM hellos) |
| **RFC** | 7761 (PIM-SM), 3973 (PIM-DM), 5015 (Bidir) |
| **"Protocol Independent"** | 기본 unicast 라우팅 테이블 사용 |

---

## 1. 한 줄 정의

라우터 간 **multicast 트래픽 전달** 프로토콜. **PI** (Protocol Independent) —
어떤 unicast 라우팅 프로토콜이든 그 위에서 동작.

---

## 2. 핵심 — RPF (Reverse Path Forwarding)

```
Multicast packet 받음 → Source IP 의 unicast 라우팅 테이블 검색
받은 인터페이스 = "Source 로 가는 인터페이스" 면 forward
아니면 폐기 (loop 방지)
```

→ unicast 의 "역방향" 으로 multicast forwarding.

### RPF Check

```
Multicast (S=10.0.0.5, G=224.1.1.1) 받음 (Eth0 에서)

→ Routing table 검색: 10.0.0.5 reach via Eth0 ?
   ✅ Forward (RPF success)
   ❌ Drop (RPF failure)
```

---

## 3. PIM 종류

### 3.1 PIM-DM (Dense Mode, RFC 3973)

**"Flood-and-Prune"**:
```
1. Source 가 보냄 → 모든 라우터에 flood
2. 가입자 없는 라우터 → Prune 메시지로 자기 끊음
3. 새 가입자 → Graft 메시지로 다시 연결
```

- 가입자 많을 때 (Dense)
- 비효율 — 전체 flood
- 거의 사용 X

### 3.2 PIM-SM (Sparse Mode, RFC 7761)

**Rendezvous Point (RP) 기반** — 가장 흔함:

```
1. 가입자 → 자기 라우터에 IGMP join (224.1.1.1)
2. 라우터 → RP 로 향한 Shared Tree (RPT) 구축
3. Source 가 보냄 → RP 까지 register 메시지
4. RP 가 Shared Tree 로 forward
5. 트래픽 양 많아지면 → Shortest Path Tree (SPT) 로 전환
```

- 가입자 적을 때 (Sparse)
- RP 가 중요 — 단일 장애점
- 대부분 환경에서 사용

### 3.3 PIM-SSM (Source-Specific Multicast)

- 232.0.0.0/8 범위
- (Source, Group) 직접 — RP 불필요
- IGMPv3 / MLDv2 의 source filtering 필요
- 인터넷 multicast 의 현실적 모델

### 3.4 PIM Bidir (Bidirectional)

- Many-to-many — 모든 source 가 보낼 수 있음
- 게임 / 회의 시스템

---

## 4. PIM-SM 동작 자세히

### 4.1 단계

```
1. RP 선출 / 설정
2. Source 가 R(P)egister → RP
   - Source 의 first-hop 라우터가 RP 에 unicast register
3. RP 가 Shared Tree 의 모든 가입자에 forward
4. 가입자 라우터 → RP 로 (*, G) Join 메시지 (RPT)
5. 트래픽 많으면 → Source 직접 (S, G) Join → SPT
6. SPT 활성 후 → RP register 중단
```

### 4.2 Tree 종류

- **(*, G) RPT** — Shared Tree, RP 중심
- **(S, G) SPT** — Source Tree, 효율적

### 4.3 SPT Switchover
- 트래픽 임계치 도달 시 RPT → SPT
- 더 짧은 경로, RP 부담 ↓

---

## 5. RP 선출

### 5.1 Static
- 모든 라우터에 명시 설정

### 5.2 Auto-RP (Cisco)
- Mapping Agent 가 RP 자동 분배

### 5.3 Bootstrap Router (BSR, RFC 5059)
- 표준 동적 선출
- 우선순위 + Hash

### 5.4 Anycast RP
- 여러 RP 가 같은 IP 광고
- HA / 부하 분산
- MSDP 로 source 정보 동기

---

## 6. MSDP — Multicast Source Discovery (RFC 3618)

여러 PIM 도메인 / RP 간 source 정보 공유:

```
RP1 (도메인 A) ↔ MSDP ↔ RP2 (도메인 B)
"내 도메인에 source X 가 그룹 G 로 보내고 있어"
```

- 옛 표준
- IPv6 는 Embedded RP 또는 PIM-SM 안의 메커니즘

---

## 7. PIM Messages

| 종류 | 용도 |
| --- | --- |
| **Hello** | 이웃 PIM 라우터 발견 |
| **Register** | Source → RP |
| **Join/Prune** | Tree 가입 / 탈퇴 |
| **Bootstrap** | RP 분배 (BSR) |
| **Candidate-RP-Advertisement** | RP 후보 광고 |
| **Assert** | LAN 의 designated forwarder 선출 |
| **Graft** | PIM-DM 의 재가입 |
| **State Refresh** | PIM-DM 의 prune 갱신 |

---

## 8. Cisco 설정 예

```
ip multicast-routing
ip pim rp-address 192.168.100.1     # Static RP

interface Gi0/0
  ip pim sparse-mode

! SSM 활성
ip pim ssm default                    # 232.0.0.0/8

! BSR
ip pim bsr-candidate Gi0/0 0
ip pim rp-candidate Gi0/0
```

### Linux FRR

```
ip multicast-routing
interface eth0
  ip pim
```

---

## 9. 디버깅

```
show ip mroute              # multicast 라우팅 테이블
show ip pim neighbor
show ip pim rp
show ip pim interface
show ip rpf 10.0.0.5         # RPF check
debug ip pim
debug ip mrouting
```

### mroute 출력 예

```
(*, 224.1.1.1), 00:01:23/00:02:34, RP 192.168.100.1, flags: SC
  Incoming interface: Gi0/0, RPF nbr 192.168.1.1
  Outgoing interface list:
    Gi0/1, Forward/Sparse, 00:01:23/00:02:34

(10.0.0.5, 224.1.1.1), 00:00:34/00:02:25, flags: T
  Incoming interface: Gi0/2, RPF nbr 192.168.2.1
  Outgoing interface list:
    Gi0/1, Forward/Sparse, 00:00:34/00:02:25
```

- `*` — Shared Tree
- `(S, G)` — Source Tree
- `T` — SPT 전환됨

---

## 10. 함정

### 함정 1 — RPF Failure
비대칭 라우팅 시 multicast drop. `mroute` static 또는 별도 unicast 경로.

### 함정 2 — RP 단일 장애
SM 의 가장 큰 위험. Anycast RP 권장.

### 함정 3 — IGMP / MLD 미설정
가입자 라우터가 IGMP 안 처리 → multicast 안 전달.

### 함정 4 — PIM DR 선출 충돌
LAN 에 여러 PIM router → DR 선출 (높은 priority / IP).

### 함정 5 — Multicast 의 인터넷 통과
대부분 ISP 가 multicast 차단. 사내망 / 직접 peering 만.

### 함정 6 — SSM 의 클라이언트 요구사항
IGMPv3 / MLDv2 필요 — 옛 OS 지원 X.

### 함정 7 — Multicast 로 큰 패킷
단편화 시 RPF 깨질 수 있음.

---

## 11. 사용 사례

### 11.1 IPTV
- 사내 / ISP 의 라이브 채널
- PIM-SM 또는 SSM

### 11.2 금융 시장 데이터
- HFT 환경 — 주식 시세 broadcast
- Multicast 의 낮은 latency

### 11.3 비디오 / 회의
- 사내 화상 (옛)
- Webex / Zoom 등 unicast 로 대체

### 11.4 라우팅 프로토콜 자체
- OSPF / RIPv2 / EIGRP / PIM Hello
- Link-local multicast (`224.0.0.x` / `ff02::x`)

---

## 12. 학습 자료

- RFC 7761 (PIM-SM), RFC 4607 (SSM), RFC 5059 (BSR)
- **Developing IP Multicast Networks** (Williamson)
- **Interdomain Multicast Routing** (Edwards)
- Cisco "PIM Configuration"

---

## 13. 관련

- [[../osi-7-layer/layer-3-network/multicast-igmp]] — IGMP / Multicast 주소
- [[routing]] — 라우팅 hub
- [[ospf]] — IGP 위 동작
