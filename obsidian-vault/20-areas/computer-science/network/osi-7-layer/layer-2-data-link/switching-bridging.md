---
title: "Switching & Bridging — L2 Switch 동작"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:05:00+09:00
tags:
  - network
  - layer-2
  - switch
  - bridge
  - cam-table
---

# Switching & Bridging — L2 Switch 동작

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Bridge / L2 Switch / CAM / Cut-through / 가상 Switch |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. Bridge → Switch 의 진화

| 시대 | 장치 | 특징 |
| --- | --- | --- |
| 1980s | Repeater / Hub | 충돌 도메인 X |
| 1990s | **Bridge** | 2-4 포트, SW 구현, 충돌 도메인 분리 |
| 1995+ | **L2 Switch** | 다중 포트 (24/48+), HW ASIC, full-duplex |
| 2000+ | **L3 Switch** | 라우팅 통합, 데이터센터 표준 |
| 2010+ | **L4-7 Switch** | LB, WAF, Application aware |

**Bridge ≈ Switch** — Switch 는 멀티 포트 + HW 가속 Bridge.

---

## 2. L2 Switch 의 3 가지 동작

### 2.1 Learning (학습)

```
입력 프레임의 Source MAC 을 보고
(MAC, Port) 쌍을 CAM table 에 저장
```

### 2.2 Forwarding (전달)

```
Destination MAC 검색:
- 있음   → 해당 포트로만 전달 (unicast)
- 없음   → 모든 포트로 flooding (unknown unicast)
- Broadcast → 모든 포트
- Multicast → IGMP snooping 이 안 켜져 있으면 모든 포트
```

### 2.3 Filtering (필터링)

같은 포트에서 온 프레임의 목적지가 같은 포트면 forward X (이미 도착).

---

## 3. CAM Table (Content-Addressable Memory)

### 3.1 구조

```
+-----+-------------+----------+--------+
| VLAN| MAC         | Port     | Aging  |
+-----+-------------+----------+--------+
| 10  | 00:1A:2B:...| Gi0/1    | 240s   |
| 10  | 00:50:56:...| Gi0/2    | 60s    |
| 20  | 00:0C:29:...| Gi0/5    | 120s   |
+-----+-------------+----------+--------+
```

### 3.2 CAM 특성

- 일반 RAM 은 주소 → 값
- **CAM** 은 값 → 주소 (역방향)
- HW 가 1 사이클에 검색
- **TCAM** (Ternary CAM) — 0/1/X (wildcard) 가능 — ACL, 라우팅

### 3.3 Aging

- 기본 5 분 (300s) — Cisco
- 마지막 사용 후 갱신 안 되면 삭제
- Topology 변화 적응

### 3.4 CAM 크기

| 등급 | 항목 수 |
| --- | --- |
| Small Office Switch | 4K-8K |
| Enterprise Switch | 16K-128K |
| Data Center Switch | 256K-1M+ |

---

## 4. Forwarding 방식

### 4.1 Store-and-Forward

```
프레임 전체 수신 → FCS 검증 → forward
```

- 장점: 손상 프레임 폐기 (CRC 검증)
- 단점: 지연 큼 (프레임 전체 대기)
- 대형 / 기업 / WAN 표준

### 4.2 Cut-through (Fragment-Free)

```
Destination MAC (14 byte) 만 읽고 즉시 forward
```

- 장점: 매우 빠름 (μs 단위)
- 단점: 손상 프레임도 통과
- HFT, 데이터센터 토르 스위치 (Cisco Nexus, Arista)

### 4.3 Fragment-free

- 첫 64 byte (충돌 가능 영역) 읽고 forward
- Cut-through 와 Store-and-Forward 의 중간

---

## 5. Switch HW 구조

```
┌────────────────────────────────────────┐
│              Switch Fabric             │  ← 모든 포트 간 패킷 전송
└────┬────┬────┬────┬────┬────┬────┬────┘
     │    │    │    │    │    │    │
   PHY  PHY  PHY  PHY  PHY  PHY  PHY     ← 각 포트
     │    │    │    │    │    │    │
   MAC  MAC  MAC  MAC  MAC  MAC  MAC
     │    │    │    │    │    │    │
   PHY-MAC-Switch Fabric 모두 ASIC
   CAM, TCAM 별도 칩 또는 통합
```

### 5.1 Switch Fabric 종류

- **Shared Bus** — 옛 (단순)
- **Crossbar** — N × N 스위치 매트릭스
- **CLOS Network** — 다단 crossbar — 대형 스위치 (Cisco Nexus, Arista 7500)
- **Shared Memory** — 작은 스위치

### 5.2 Throughput / PPS

- **Throughput** — bps (각 포트 합)
- **PPS** (Packets Per Second) — packet/s
- "Wire-speed" / "Line-rate" — 모든 포트 최대 동시

```
64-port 100GbE = 6.4 Tbps
PPS = 6.4e12 / (64 × 8) bit = 9.5 Gpps (최소 64-byte frame 기준)
```

---

## 6. Spanning Tree (STP) — 루프 방지

자세히 → [[vlan-stp-bonding]]

### 핵심
- Switch 루프 → 무한 broadcast → "Broadcast Storm"
- STP 가 일부 포트 BLOCK 해 트리 형성

---

## 7. VLAN — 가상 LAN

자세히 → [[vlan-stp-bonding]]

### 핵심
- 같은 Switch 의 포트들을 논리 그룹
- 802.1Q tag 로 trunk
- Broadcast 도메인 분리

---

## 8. Link Aggregation (LAG / LACP)

자세히 → [[vlan-stp-bonding]]

- 여러 물리 링크 → 1 논리 링크
- 대역폭 ↑ + 이중화
- 802.3ad / 802.1AX

---

## 9. Port Mirroring (SPAN / Monitor)

```
Source Port → 정상 트래픽
Mirror Port → 같은 트래픽 복제 (분석용)
```

- Wireshark / IDS / IPS 가 수신
- **SPAN** (Cisco) / Port Mirror (general)
- **RSPAN** — 원격 (VLAN 으로)
- **ERSPAN** — IP 캡슐화 (어디서나)

---

## 10. Port Security

CAM Overflow / MAC Spoofing 방어:

```
switchport port-security maximum 2
switchport port-security violation shutdown
switchport port-security mac-address sticky
```

- 포트당 학습 MAC 수 제한
- 위반 시: protect / restrict / shutdown
- 802.1X 와 결합 — 인증된 사용자만

---

## 11. 가상 스위치 (Virtual Switch)

VM / 컨테이너 환경:

### 11.1 종류

- **vSphere vSwitch / DVS** — VMware
- **Hyper-V Virtual Switch** — Microsoft
- **OVS (Open vSwitch)** — Linux, OpenStack, Kubernetes
- **Linux Bridge** — `bridge` 명령, kernel module
- **MACVLAN / IPVLAN** — Docker 네트워크

### 11.2 OVS

```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 eth0
ovs-ofctl dump-flows br0
```

- OpenFlow 지원 — SDN 의 데이터플레인
- VLAN, GRE, VXLAN 캡슐화
- Kubernetes (Cilium / Calico) 의 데이터플레인

### 11.3 SmartNIC offload
- OVS rule 을 NIC HW 로 — bare-metal 성능

---

## 12. 충돌 도메인 / 브로드캐스트 도메인 비교

```
─ Hub ─
충돌 도메인 1, broadcast 도메인 1
모든 포트 → 같은 도메인

─ Bridge / L2 Switch ─
충돌 도메인 N (포트 별)
broadcast 도메인 1 (VLAN 안)

─ Router / L3 Switch ─
충돌 도메인 N
broadcast 도메인 N (서브넷 별)
```

---

## 13. Switch 종류 정리

| 종류 | 영역 | 예 |
| --- | --- | --- |
| **Unmanaged** | 가정 / 작은 사무실 | TP-Link 5-port |
| **Managed** | 기업 / 데이터센터 | Cisco Catalyst, Arista 7050 |
| **L3 Switch** | 라우팅 통합 | Cisco Catalyst 9300, Arista 7280 |
| **Spine Switch** | 데이터센터 백본 | Arista 7500R3, Cisco Nexus 9500 |
| **Leaf / ToR** | 서버 직접 연결 | Arista 7050, Cisco Nexus 93180 |
| **Edge Switch** | 사용자 연결 | PoE, 802.1X |

### CLOS Spine-Leaf 토폴로지

```
   Spine    Spine    Spine    Spine
     │        │        │        │
   ──┼────────┼────────┼────────┼──
     │        │        │        │
   Leaf     Leaf     Leaf     Leaf
   ───      ───      ───      ───
   Server   Server   Server   Server
```

- 모든 leaf 가 모든 spine 에 연결
- 어디서 어디로든 2 hops (deterministic latency)
- 데이터센터 표준

---

## 14. CLI 예제 (Cisco)

```
# CAM 확인
show mac address-table

# 특정 MAC 어디?
show mac address-table address 00:1A:2B:...

# 학습 비활성화
no mac address-table learning vlan 10

# Port Security
interface Gi0/1
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
```

---

## 15. 함정

### 함정 1 — Switch loop
STP 비활성 / 잘못 설정 시 broadcast storm. STP 항상 켜기.

### 함정 2 — CAM Overflow
악의적 MAC flooding 으로 hub 처럼. Port Security 필요.

### 함정 3 — Trunk 와 Access 혼동
Trunk = 여러 VLAN, Access = 1 VLAN. 잘못 설정 시 VLAN hopping.

### 함정 4 — Cut-through 의 손상 프레임
다음 hop 에서 폐기 — 효율 낮을 수 있음.

### 함정 5 — Switch 의 MTU 일관성
mismatch 시 jumbo frame drop.

### 함정 6 — Storm Control 미설정
악의적 / 버그성 트래픽이 모든 포트 마비.

---

## 16. 학습 자료

- Cisco "CCNP Enterprise" 학습서
- **Computer Networking: Top-Down Approach** Ch. 6
- Arista EOS / Cisco IOS 매뉴얼
- "Data Center Switch Architecture" (Arista 백서)

---

## 17. 관련

- [[ethernet-frame]] — Switch 가 처리하는 프레임
- [[mac-address]] — CAM 의 키
- [[vlan-stp-bonding]] — VLAN / STP / LAG
- [[layer-2-data-link]] — 상위
