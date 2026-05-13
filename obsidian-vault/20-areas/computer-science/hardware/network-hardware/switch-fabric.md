---
title: "스위치·패브릭 (Switch / Fabric)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, switch, router, fabric, spine-leaf, fat-tree, dragonfly, asic, tomahawk, sdn, p4, optical, cpo, pam4, ecmp]
---

# 스위치·패브릭

**[[network-hardware|↑ 네트워크 하드웨어]]**

> 데이터센터 / 캠퍼스 / 통신사의 packet forwarding 백본. ASIC 의 capacity 가 시대마다 2 배씩 늘면서 데이터센터 토폴로지가 진화.

## 1. L2 / L3 차이

### Ethernet Frame (L2)

```
┌──────────────┬──────────────┬─────┬──────────┬─────────┐
│ Dst MAC      │ Src MAC      │ Type│ Payload  │ FCS     │
│ 6 byte       │ 6 byte       │ 2 B │ ~1500 B  │ 4 byte  │
└──────────────┴──────────────┴─────┴──────────┴─────────┘
```

### IP Packet (L3)

```
┌──────────────┬──────────────┬─────┬─────────┐
│ Src IP       │ Dst IP       │ proto│ Payload │
│ 4 byte (v4)  │ 4 byte       │ 1 B │         │
└──────────────┴──────────────┴─────┴─────────┘
```

| 계층 | 헤더 | 의미 |
| --- | --- | --- |
| **L2 스위치** | Ethernet (MAC) | 같은 subnet 안 frame forwarding. MAC 학습 + flooding. |
| **L3 스위치 / 라우터** | IP | subnet 간 라우팅. routing table / OSPF / BGP. |

## 2. L2 스위치 동작 — MAC Learning + Flooding

### MAC 학습
- frame 도착 → src MAC + 도착 port 를 MAC table 에 저장.
- table aging (300 초 기본).

### Frame forwarding
- dst MAC 이 table 에 있으면 그 port 만.
- table 에 없으면 **flooding** (수신 port 외 모든 port).
- broadcast (FF:FF:FF:FF:FF:FF) 는 항상 flooding.

### VLAN
- 802.1Q tag (4 byte) 를 ethernet frame 에 삽입.
- 같은 물리 스위치를 여러 logical L2 로 분리.
- VLAN ID 12-bit → 4094 VLANs.

### STP / RSTP / MSTP
- L2 loop 방지.
- spanning tree 만들어 일부 link blocking.
- RSTP (Rapid) = 50 ms 수렴.
- modern 데이터센터는 STP 보다 L3 fabric 선호.

## 3. L3 라우터 동작

### Routing Table
- destination prefix → next-hop / outgoing interface.
- longest prefix match.
- 표준 라우팅 프로토콜: OSPF / BGP / IS-IS / RIP.

### ECMP (Equal-Cost Multi-Path)
- 같은 cost 의 여러 path 가 있으면 5-tuple hash 로 분산.
- 데이터센터 spine-leaf 의 핵심.

### Forwarding Plane vs Control Plane
- **Control plane** — routing protocol (OSPF, BGP) 실행 + 결정.
- **Data plane** — packet 을 lookup table 따라 forwarding.
- 분리 → SDN.

## 4. 스위치 ASIC — 단일 칩 capacity

| 칩 | 출시 | 단일 chip BW | 포트 |
| --- | --- | --- | --- |
| Broadcom Trident 2 | 2013 | 1.28 Tbps | 32×40G |
| Broadcom Tomahawk 1 | 2014 | 3.2 Tbps | 32×100G |
| Broadcom Tomahawk 2 | 2016 | 6.4 Tbps | 64×100G |
| Broadcom Tomahawk 3 | 2017 | 12.8 Tbps | 32×400G |
| Broadcom Tomahawk 4 | 2019 | 25.6 Tbps | 32×800G |
| **Broadcom Tomahawk 5** | 2022 | **51.2 Tbps** | 64×800G |
| **Broadcom Tomahawk 6** | 2025 (예정) | **102.4 Tbps** | 64×1.6T |
| **NVIDIA Spectrum-4** | 2022 | 51.2 Tbps | 64×800G |
| **NVIDIA Quantum-2 (InfiniBand)** | 2021 | 25.6 Tbps | 64×400G NDR |
| **Cisco Silicon One G200** | 2023 | 51.2 Tbps | — |
| **Marvell Teralynx 10** | 2024 | 51.2 Tbps | — |

→ 단일 1U 박스 안 64×800G 포트 = data center ToR 1 대 충분.

## 5. 데이터센터 토폴로지

### Spine-Leaf — 현 표준

```
   Spine 1   Spine 2   Spine 3   Spine 4
     │  │  │   │  │  │   │  │  │   │  │  │
     ▼  ▼  ▼   ▼  ▼  ▼   ▼  ▼  ▼   ▼  ▼  ▼
   Leaf 1  Leaf 2  Leaf 3  Leaf 4  Leaf 5
     │       │       │       │       │
   Hosts   Hosts   Hosts   Hosts   Hosts
```

- 모든 leaf 가 모든 spine 에 직결.
- **동일 latency**: 어떤 두 호스트 간이라도 leaf → spine → leaf 의 3-hop.
- **non-blocking**: spine 대역폭 합 = leaf 의 다운링크 합.
- L3 ECMP 가 spine 사이 부하 분산.

### Fat-tree
- 트리 + 위로 갈수록 대역폭 ↑.
- spine-leaf 의 일반화.

### Dragonfly / Dragonfly+
- HPC / AI 학습 슈퍼컴.
- **group** = router 여러 개 fully connected.
- **global link** = group 사이.
- ML 학습의 all-reduce 트래픽에 최적.
- HPE Slingshot, Cray, Frontier 슈퍼컴.

### Torus / 3D Torus
- 옛 supercomputer (Cray T3D, IBM Blue Gene).
- 각 노드가 ± X/Y/Z 6 방향 이웃과 직결.
- 현 trend 는 Dragonfly / Fat-tree.

## 6. 데이터센터 oversubscription

### Oversubscription ratio
- 다운링크 BW : 업링크 BW.
- 1:1 = non-blocking (모든 호스트가 동시 line rate).
- 4:1 = 다운링크 4× > 업링크. 비용 ↓, traffic 패턴 따라 가능.

### AI 학습 클러스터
- 1:1 non-blocking 필수.
- 모든 GPU 가 동시 AllReduce 트래픽 발생.

### 일반 데이터센터
- 3:1 또는 4:1 일반.
- 비용 절감.

## 7. ECMP — Equal-Cost Multi-Path

### 동작
- 같은 cost 라우팅 path 여러 개 있으면 hash 로 분산.
- per-flow hash (같은 flow 는 같은 path).
- 5-tuple (src IP / dst IP / src port / dst port / proto).

### Hash polarization 문제
- 모든 hop 의 hash 함수 / key 가 같으면 일부 path 만 사용.
- 해결: 각 hop 의 hash key 를 다르게 (randomize).

### Multi-path TCP / Per-packet ECMP
- per-packet ECMP = packet 마다 다른 path → out-of-order packet → TCP 성능 ↓.
- 일반 ECMP 는 flow 단위.

## 8. Adaptive Routing

- 정적 ECMP 의 한계 — 일부 path 혼잡해도 계속 같은 hash 사용.
- adaptive routing — 혼잡 path 회피해서 동적 선택.
- packet 순서 깨질 수 있음 → reorder buffer 필요.
- InfiniBand / HPE Slingshot 의 강점.

## 9. SDN (Software Defined Networking)

### 분리된 컨트롤 + 데이터 plane

```
   Controller (소프트웨어)
        │
        │ OpenFlow / P4Runtime
        ▼
   Switch hardware (데이터 플레인만)
```

### OpenFlow
- 2008 Stanford. SDN 의 시작.
- 컨트롤러가 switch flow table 직접 관리.
- Google 의 B4 WAN backbone 이 OpenFlow 기반.

### P4 (Programming Protocol-independent Packet Processors)
- 2014 Stanford / Princeton.
- programmable 데이터 플레인 — 직접 packet 처리 logic 작성.
- Tofino 칩 (Intel, 단종) 이 첫 P4 ASIC.
- 보안 / load balancer / 통신사 가속.

### eBPF / XDP — Host-side SDN
- 호스트 단의 packet 처리 programmable.
- Cilium 같은 클러스터 networking 에 사용.

## 10. 광 / 트랜시버

| 폼팩터 | 속도 | 거리 (SR / LR / ZR) | 대표 |
| --- | --- | --- | --- |
| **SFP+** | 10 G | 10 km / 80 km | 일반 |
| **SFP28** | 25 G | 100m / 10 km | |
| **QSFP+** | 40 G (4×10) | 100m-10km | |
| **QSFP28** | 100 G (4×25) | 100m-80km | 일반 |
| **QSFP-DD** | 200 / 400 G (8×25/50) | 500m-10km | 신규 |
| **OSFP** | 400 / 800 G (8×50/100) | 500m-10km | DGX H100 |
| **OSFP-XD** | 1.6 T | (예정) | 차세대 |

### PAM4 — 변조

- NRZ (Non-Return to Zero): 1 심볼 = 1 bit.
- PAM4: 4 amplitude level = 1 심볼 = 2 bit.
- 같은 baud rate (32 GBaud) 로 2 배 throughput.
- 단점: SNR margin 작음 → FEC 필요.
- 50G / 100G / 200G per lane 의 핵심.

### 차세대 — PAM6 / coherent

- PAM6 (1 심볼 ~2.5 bit) → 200 G per lane.
- Coherent optics — DSP 기반, long-haul 표준.

## 11. CPO (Co-Packaged Optics)

광 트랜시버를 ASIC 옆 패키지에 통합.

### 동기
- 800 GbE / 1.6 T 트랜시버의 SerDes 가 큰 발열원.
- 전기 신호가 PCB 위 거리 길수록 손실 ↑.
- CPO = 트랜시버를 ASIC 의 mm 거리에.

### 효과
- 전력 절감 30-50%.
- BW 한계 ↑.

### 출시
- 2024-2026 본격. Broadcom / NVIDIA / Intel 모두 작업 중.

## 12. DAC / AOC

### DAC (Direct Attached Copper)
- 짧은 거리 (3-5 m) 의 동선 케이블.
- 광 트랜시버보다 싸고 latency 낮음.
- rack 내부 / 인접 rack 에 일반.

### AOC (Active Optical Cable)
- 광 + 양 끝에 전기-광 변환 칩.
- 100 m 까지.
- 광 트랜시버 분리 불필요.

## 13. 무선 / Wi-Fi / Bluetooth

자세히는 [[network-hardware]] hub.

### Wi-Fi 진화
| 표준 | 출시 | 대역 | 속도 |
| --- | --- | --- | --- |
| Wi-Fi 5 (802.11ac) | 2014 | 5 | 3.5 Gbps |
| Wi-Fi 6 (802.11ax) | 2019 | 2.4/5 | 9.6 Gbps |
| Wi-Fi 6E | 2020 | 2.4/5/6 | 9.6 Gbps |
| **Wi-Fi 7 (802.11be)** | 2024 | 2.4/5/6 | 46 Gbps |
| Wi-Fi 8 | 2028~ | (개발) | 100 Gbps+ |

### Wi-Fi 7 의 핵심 — MLO
- **MLO (Multi-Link Operation)** — 여러 대역 (2.4 / 5 / 6 GHz) 동시 사용.
- **4096-QAM** — 더 dense modulation.
- **320 MHz 채널** — 더 wide bandwidth.

## 14. 데이터센터 가동 사례

### Meta / Facebook
- 4 계층 fat-tree.
- 400 GbE leaf, 800 GbE spine.
- Backbone Express 의 large-scale ECMP.

### Google
- Jupiter fabric.
- 자체 디자인 스위치 (Pluto OS).
- B4 WAN backbone — SDN + OpenFlow.

### AWS
- Custom switch ASIC.
- 자체 운영 OS.
- NIC + DPU (Nitro).

### AI 학습 클러스터 (NVIDIA DGX SuperPOD)
- **InfiniBand 400G NDR** (Quantum-2) 또는 **800G XDR** (Quantum-3).
- non-blocking fat-tree.
- AllReduce 트래픽이 클러스터 internal BW 의 80%+.

## 15. NOS (Network Operating System)

### Open / Custom
- **SONiC** (Microsoft 오픈소스) — 가장 큰 open NOS. Azure 사용.
- **Cumulus Linux** (NVIDIA 인수) — Linux 기반.
- **DENT** — Linux Foundation.
- **OpenSwitch** — 다중 vendor.

### Vendor
- **Cisco IOS-XR / NX-OS**.
- **Juniper Junos**.
- **Arista EOS** — Linux 기반.

### 추세
- Open 화. switch HW + open NOS 분리.
- 자체 운영 (Meta / Google) 또는 SONiC 채택.

## 16. SRv6 / Segment Routing — modern WAN

### 개념
- IPv6 헤더에 segment list (router 경로) 박음.
- network 가 stateless — 각 router 가 다음 segment 만 보고 forwarding.

### 사용
- 통신사 backbone.
- SD-WAN.
- Network slicing (5G).

## 17. 버퍼 / 큐 / QoS

### Switch ASIC 의 버퍼
- 공유 버퍼 (shared) vs port 별 dedicated.
- 단위: MB.
- microburst (μs 단위 폭주) 대응에 중요.

### Buffer-bloat
- 큰 버퍼가 latency tail 늘림.
- AQM (Active Queue Management) 로 packet 적극 drop.
- **CoDel**, **PIE**, **FQ-CoDel**.

### ECN (Explicit Congestion Notification)
- packet drop 대신 ECN bit 설정.
- TCP 가 속도 감소.
- RoCE 의 DC-QCN 도 ECN 기반.

### PFC (Priority Flow Control, 802.1Qbb)
- 한 priority 의 packet 만 pause.
- RoCE 의 lossless 보장.
- PFC storm 위험 (한 호스트 PAUSE 가 전파).

## 18. 모니터링

### sFlow / IPFIX / NetFlow
- packet 일부 sampling → flow 통계.
- traffic 패턴 분석.

### Telemetry
- gRPC / Telemetry / IPFIX-PSAMP.
- ASIC 의 buffer 사용량, drop counter 등 streaming.

### INT (In-band Network Telemetry)
- packet 안에 metadata (hop 별 latency, queue 깊이) 박음.
- end-to-end 경로 추적.
- P4 칩 활용.

## 19. 진단

```bash
# 호스트 측에서 ECMP path 확인
mtr -nrwzc 30 destination_ip          # multi-hop path
traceroute -n -m 30 destination_ip

# Linux router 의 라우팅
ip route show
ip route get 8.8.8.8

# 호스트의 PFC / ECN
sudo dcb pfc show dev eth0
sudo ethtool -S eth0 | grep -E 'pause|ecn'

# Mellanox switch (NVOS)
nv show interface
nv show interface counters
nv show bgp neighbor

# Cisco Nexus
show interface counters
show ip route
show port-channel summary
```

## 20. 함정

1. **ECMP hash polarization** — 모든 hop hash key 같으면 일부 path 만 사용.
2. **MTU mismatch** — 한 hop 만 1500 이면 9000 jumbo 가 fragment → throughput 폭락.
3. **L2 broadcast storm** — STP / RSTP 없이 loop 만들면 망 마비.
4. **VLAN 4094 한계** — 클라우드 멀티 테넌트는 VXLAN / Geneve 필요.
5. **광 단자 청결** — 먼지로 BER (Bit Error Rate) ↑.
6. **버퍼 부족 → microburst drop** — TCP 의 burst 트래픽 처리에 큰 buffer ASIC 권장.
7. **PFC storm** — 한 호스트의 NIC 가 끊임없이 PAUSE → 인접 호스트도 멈춤.
8. **InfiniBand 와 Ethernet 혼동** — 같은 QSFP-DD 포트지만 fabric 다름. switch mode 일치 필수.
9. **CPO 의 hot-swap 불가** — 트랜시버 fail 시 ASIC 까지 교체.
10. **SDN controller SPOF** — controller 죽으면 forwarding 결정 못 함. HA 필수.
11. **Bufferbloat** — large buffer 가 latency 늘림. AQM 켜기.
12. **vendor-specific SNMP MIB** — 자동화 시 vendor 변경 어려움. open standard (gNMI) 선호.

## 21. 관련

- [[network-hardware]]
- [[nic-offload]]
- [[rdma-infiniband]] — lossless fabric, IB
- [[../../network/network]] — IP / TCP / BGP / OSPF
- [[../../distributed-systems/distributed-systems]] — Spine-Leaf / Fat-tree 의 분산 시스템 영향
