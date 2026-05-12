---
title: "VLAN, STP, Link Aggregation"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:10:00+09:00
tags:
  - network
  - layer-2
  - vlan
  - stp
  - lacp
---

# VLAN, STP, Link Aggregation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 802.1Q VLAN, 802.1D/w/s STP, 802.3ad LACP |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

# Part A. VLAN — Virtual LAN

## A.1 왜 VLAN 인가

물리적으로 같은 Switch 위 → 논리적으로 분리된 LAN.

목적:
- **Broadcast 도메인 분리** — 효율 / 보안
- **부서 분리** — Sales / Engineering / HR
- **Trust 경계** — DMZ / 내부
- **이동성** — 사용자 이동해도 같은 VLAN

## A.2 802.1Q 표준 (1998)

이전엔 vendor 별 (Cisco ISL) → 표준화.

### Tag 위치
Source MAC 뒤, EtherType 앞:

```
[Dest][Src][802.1Q tag 4 byte][EtherType][Data][FCS]
```

### Tag 구조 (4 byte)

```
┌──────────────┬──────┬───┬────────┐
│ TPID 0x8100  │ PCP  │DEI│ VID    │
│   16 bit     │  3   │ 1 │ 12 bit │
└──────────────┴──────┴───┴────────┘
```

- **TPID** = 0x8100
- **PCP** (Priority) — 0-7 (QoS)
- **DEI** (Drop Eligible) — 혼잡 시 폐기 가능
- **VID** — VLAN ID (1-4094, 0/4095 예약)

## A.3 포트 모드

| 모드 | 동작 |
| --- | --- |
| **Access** | 1 VLAN, untagged in / out |
| **Trunk** | 여러 VLAN, tagged (Native VLAN 만 untagged) |
| **Hybrid** | 일부 VLAN tagged, 일부 untagged (사용 X) |
| **General** | HP / Aruba 표기, Hybrid 같음 |

## A.4 Access vs Trunk 예시

```
PC ── [Gi0/1 Access VLAN 10] ── Switch ── [Gi0/24 Trunk 10,20,30] ── Switch2
                                                 ↑
                                          tagged 10/20/30
                                          + untagged native VLAN (보통 1)
```

## A.5 Native VLAN

- Trunk 의 untagged 트래픽이 속하는 VLAN
- 기본 1 (Cisco) — 보안상 변경 권장
- 양 끝 Native VLAN 일치 필수

## A.6 Voice VLAN

- 일반 PC + IP Phone → 한 포트
- IP Phone 이 자기 트래픽 VLAN tag, PC 트래픽 그대로
- LLDP-MED 로 자동 협상

## A.7 QinQ (802.1ad)

```
[Customer VLAN][Service VLAN][Data]
```

- ISP 가 고객 VLAN 위에 자기 VLAN
- TPID = 0x88A8 (S-TPID)
- 통신사 메트로 이더넷

## A.8 VLAN 트래픽 라우팅

다른 VLAN 간 통신은 L3 필요:
- **Router on a Stick** — Switch 의 Trunk + Router 서브인터페이스
- **L3 Switch** — VLAN Interface (SVI), 가장 흔함

## A.9 Private VLAN (PVLAN)

VLAN 안의 격리:
- **Primary VLAN**
  - **Isolated VLAN** — 서로 통신 X
  - **Community VLAN** — 같은 community 끼리만
  - **Promiscuous** — 모두와 통신 (게이트웨이)

호텔 / 공유 호스팅에서 사용.

## A.10 함정

- **VLAN Hopping** — Double tag (802.1Q + 추가) 로 다른 VLAN 침투. Native VLAN 변경 / DTP 비활성으로 방어.
- **Native VLAN 불일치** — STP 깨짐 / 트래픽 누수.
- **DTP** (Dynamic Trunking Protocol) — 보안상 명시적 Trunk/Access 권장.

---

# Part B. STP — Spanning Tree Protocol

## B.1 왜 STP 인가

루프 = 죽음:

```
Switch A ─── Switch B
   │             │
   └─── Switch C ┘

Broadcast → 무한 순환 → Storm
MAC table flapping
```

L3 IP 는 TTL 로 끝나지만 L2 Ethernet 은 TTL 없음 → 영원히 순환.

## B.2 STP 동작 (Radia Perlman 1985, 802.1D)

```
1. 모든 Switch 가 BPDU 주고받음 (매 2초)
2. 가장 낮은 Bridge ID 가 Root Bridge 선출
3. 각 Switch 가 Root 로 가는 최소 경로 선택 (Root Port)
4. 각 세그먼트에 Designated Port 하나
5. 나머지 포트 = BLOCKING — 트리 형성
```

## B.3 Bridge ID

```
[Priority 16 bit][MAC 48 bit]
Priority 기본 32768, 4096 단위 변경
```

낮을수록 우선. Priority 같으면 MAC 낮은 것.

## B.4 STP 포트 상태 (오리지널 802.1D)

| 상태 | 시간 | 동작 |
| --- | --- | --- |
| **Disabled** | — | 비활성 |
| **Blocking** | 20s (Max Age) | BPDU 수신만 |
| **Listening** | 15s (Fwd Delay) | BPDU 송수신, 학습 X |
| **Learning** | 15s | MAC 학습, forward X |
| **Forwarding** | — | 정상 |

→ 전환 시간 **30-50 초** — 너무 느림.

## B.5 BPDU (Bridge Protocol Data Unit)

```
Destination MAC: 01:80:C2:00:00:00 (STP)
LLC SAP: 0x42

[Protocol ID][Version][BPDU Type][Flags][Root ID][Root Path Cost]
[Bridge ID][Port ID][Message Age][Max Age][Hello Time][Forward Delay]
```

## B.6 RSTP (Rapid STP, 802.1w, 2001)

- 전환 시간 **수 초** 로 단축
- 포트 역할: Root / Designated / **Alternate** / **Backup**
- Edge port (Portfast) — 즉시 Forwarding
- Cisco 모던 표준

### 포트 상태 (RSTP)

| 상태 | 동작 |
| --- | --- |
| Discarding | Block + Listening 통합 |
| Learning | MAC 학습 |
| Forwarding | 정상 |

## B.7 MSTP (Multiple STP, 802.1s, 2002)

- VLAN 그룹별 다른 STP 인스턴스
- 부하 분산 (VLAN 10-19 는 A 가 root, 20-29 는 B 가 root)

## B.8 PVST+ / Rapid-PVST+ (Cisco)

- VLAN 마다 별도 STP 인스턴스
- 부하 분산
- Cisco proprietary

## B.9 BPDU Guard / Root Guard

- **BPDU Guard** — Edge 포트가 BPDU 받으면 err-disable (사용자가 Switch 꽂은 경우)
- **Root Guard** — 신뢰할 수 없는 곳에서 better Root BPDU 받으면 차단
- **Loop Guard** — BPDU 안 오면 BLOCK

## B.10 함정

- **STP 꺼두기** — 절대 X. 루프 시 모든 사용자 마비.
- **Root 선출 무작위** — 작은 office Switch 가 Root 되면 dc 트래픽 비효율. 명시적 priority.
- **Convergence 시간 무시** — RSTP / MSTP 사용.
- **TCN (Topology Change)** — 자주 발생 시 MAC table flush 빈번.

---

# Part C. Link Aggregation (LAG / LACP)

## C.1 무엇

여러 물리 링크 → 1 논리 링크.

```
Switch A ═══════════ Switch B  (4 × 10G = 40G 논리)
```

목적:
- **대역폭 ↑** — N × 단일 링크
- **이중화** — 한 링크 죽어도 OK
- **부하 분산**

## C.2 표준

- **802.3ad / 802.1AX** — LACP (Link Aggregation Control Protocol)
- Cisco 의 **PAgP** — proprietary, 사장
- **Static** — 협상 없이 그냥 묶음 (위험)

## C.3 LACP 동작

1. 양쪽이 LACPDU 교환 (1 초 / 30 초)
2. 협상 — 양쪽 모두 OK 면 활성화
3. 한 링크 실패 시 자동 제외

### Mode

- **Active** — LACP 시작
- **Passive** — 받기만
- 양쪽 모두 Passive 면 협상 X

## C.4 Hash 알고리즘 (부하 분산)

한 흐름은 항상 같은 링크 (순서 보장):

| Hash 입력 | 효과 |
| --- | --- |
| Src/Dst MAC | L2 만 |
| Src/Dst IP | L3 |
| Src/Dst Port | L4 (더 균일) |
| 5-tuple | 가장 균일 |

### 부하 불균형 함정

같은 IP/Port 사용 시 (1 백업 스트림 등) 다른 링크로 분산 X → 한 링크만 폭주.

## C.5 MC-LAG (Multi-Chassis LAG)

```
        Server
       /      \
      eth0    eth1
       │       │
   Switch A ─ Switch B   ← 두 Switch 가 한 LAG 처럼
```

서버 입장에선 한 LAG 인데 실제로 두 Switch 에 연결 — 더 강한 이중화.

- Cisco vPC (virtual Port Channel)
- Arista MLAG
- Juniper Virtual Chassis

## C.6 Linux Bonding

```bash
modprobe bonding
echo "+bond0" > /sys/class/net/bonding_masters
echo 802.3ad > /sys/class/net/bond0/bonding/mode
echo eth0 > /sys/class/net/bond0/bonding/slaves
echo eth1 > /sys/class/net/bond0/bonding/slaves
```

### Linux bonding 모드

| 모드 | 설명 |
| --- | --- |
| 0 (balance-rr) | 라운드 로빈 — 순서 깨짐 가능 |
| 1 (active-backup) | 1 active, 나머지 standby |
| 2 (balance-xor) | XOR hash |
| 3 (broadcast) | 모든 슬레이브로 |
| 4 (802.3ad) | LACP |
| 5 (balance-tlb) | 송신 부하 분산 |
| 6 (balance-alb) | 송수신 모두 |

## C.7 NIC Teaming (Windows / Hyper-V)

비슷한 개념. PowerShell:

```powershell
New-NetLbfoTeam -Name "Team1" -TeamMembers eth0,eth1 -TeamingMode Lacp
```

## C.8 함정

- **양 쪽 LACP 설정 안 함** — Static + LACP 하면 무한 flap.
- **Mode 불일치** — 한쪽 802.3ad, 한쪽 active-backup → 동작 X.
- **Hash 부적절** — 단일 흐름이면 LAG 무의미.
- **Cable / SFP 미스매치** — 일부 슬레이브가 다른 속도 / duplex 면 비활성.

---

# Part D. 종합

## D.1 함께 쓰는 예 (데이터센터)

```
Server ─ LACP ─ Leaf Switch ─ MC-LAG ─ Spine Switch
              │                         │
              VLAN trunk                VLAN trunk
              802.1Q tag                802.1Q tag
              MSTP                      MSTP
              PFC (QoS)                 PFC
```

## D.2 함께 쓰는 예 (캠퍼스)

```
PC ─ Access port (VLAN 10) ─ Edge Switch ─ Trunk (VLAN 10/20/30) ─ Distribution L3 Switch
                                                                      │
                                                                      Inter-VLAN routing
                                                                      ┴
                                                                   Core Switch ─ Internet
```

## D.3 학습 자료

- IEEE 802.1Q (VLAN), 802.1D/w/s (STP), 802.3ad (LACP)
- Cisco CCNP / CCIE 가이드
- Arista MLAG 백서
- Radia Perlman — "Interconnections: Bridges, Routers, Switches"

## D.4 관련

- [[ethernet-frame]] — 802.1Q tag
- [[switching-bridging]] — Switch 동작
- [[../layer-3-network/layer-3-network]] — 라우팅 (L3 Switch)
- [[layer-2-data-link]] — 상위
