---
title: "스위치·패브릭 (Switch / Fabric)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, switch, router, fabric, spine-leaf, fat-tree, dragonfly, asic, tomahawk]
---

# 스위치·패브릭

**[[network-hardware|↑ 네트워크 하드웨어]]**

## 1. L2 / L3 차이

| 계층 | 헤더 | 의미 |
| --- | --- | --- |
| **L2 스위치** | Ethernet (MAC) | 같은 subnet 안 frame forwarding. MAC 학습 + flooding. |
| **L3 스위치 / 라우터** | IP | subnet 간 라우팅. routing table / OSPF / BGP. |

현대 데이터센터 ToR (Top of Rack) = L3 스위치 (Spine-Leaf 의 leaf).

## 2. 스위치 ASIC

| 칩 | 칩셋 | 단일 chip BW |
| --- | --- | --- |
| **Broadcom Tomahawk 5** | merchant silicon | **51.2 Tbps** (64×800G) |
| **Broadcom Tomahawk 6** (예정) | | 102.4 Tbps |
| **NVIDIA Spectrum-4** | merchant | 51.2 Tbps |
| **NVIDIA Quantum-2** (InfiniBand) | | 25.6 Tbps |
| **Cisco Silicon One G200** | | 51.2 Tbps |
| **Marvell Teralynx 10** | | 51.2 Tbps |

단일 chip 51.2 Tbps 가 1U 박스 안에 들어감 → 데이터센터 ToR 1 대 = 64×800G 포트.

## 3. 데이터센터 토폴로지

### Spine-Leaf

```
   Spine 1   Spine 2   Spine 3   Spine 4
     │  │  │   │  │  │   │  │  │   │  │  │
     ▼  ▼  ▼   ▼  ▼  ▼   ▼  ▼  ▼   ▼  ▼  ▼
   Leaf 1  Leaf 2  Leaf 3  Leaf 4  Leaf 5  ...
     │       │       │       │       │
   Hosts   Hosts   Hosts   Hosts   Hosts
```

- 모든 leaf 가 모든 spine 에 직결.
- **동일 latency**: 어떤 두 호스트 간이라도 leaf → spine → leaf 의 3-hop.
- **non-blocking**: spine 대역폭 합 = leaf 의 다운링크 합.

### Fat-tree

- 트리 + 위로 갈수록 대역폭 ↑.
- Spine-leaf 의 일반화.

### Dragonfly / Dragonfly+

- HPC / AI 학습 슈퍼컴.
- group 내부는 fully-connected, group 사이는 global link.
- ML 학습의 all-reduce 트래픽에 최적.

### Torus (3D Torus)

- 옛 supercomputer (Cray, IBM Blue Gene).
- 각 노드가 ± X/Y/Z 6 방향 이웃과 직결.
- 현 trend 는 Dragonfly.

## 4. 라우팅 / 부하 분산

### ECMP (Equal-Cost Multipath)
- 같은 cost 의 여러 path 가 있으면 5-tuple hash 로 분산.
- spine-leaf 의 핵심.

### Adaptive routing
- 혼잡 path 회피해서 dynamic 으로 선택.
- InfiniBand / Slingshot 의 강점.

### NSH / VXLAN / Geneve
- 오버레이 (가상 네트워크) 헤더 캡슐화.
- 클라우드 멀티 테넌트.

## 5. SDN (Software Defined Networking)

- 컨트롤 플레인 (라우팅 결정) 과 데이터 플레인 (실 패킷 forwarding) 분리.
- **OpenFlow** — 옛 표준.
- **P4** — programmable 데이터 플레인. Tofino 칩.
- **eBPF/XDP** — 호스트측 SDN.

## 6. 광 / 트랜시버

| 폼팩터 | 속도 | 거리 | 대표 |
| --- | --- | --- | --- |
| **SFP+** | 10 G | 10 km (SR)~80 km (ZR) | 일반 |
| **SFP28** | 25 G | 10 km~40 km | |
| **QSFP+** | 40 G | 100 m (SR)~10 km | |
| **QSFP28** | 100 G | 100 m~80 km | |
| **QSFP-DD** | 200/400 G | 500 m~10 km | |
| **OSFP** | 400/800 G | 500 m~10 km | DGX H100 등 |

**PAM4** = 1 심볼 2 bit. 50/100/200 G per lane 의 핵심.

**CPO (Co-Packaged Optics)** = 광 트랜시버를 ASIC 옆 패키지에 통합. 전력 절감 (transceiver 의 SerDes 가 가장 큰 발열원).

## 7. 데이터센터 가동 사례

- Meta / AWS / Google 의 데이터센터 = 100~400 GbE leaf, 400/800 GbE spine.
- AI 학습 클러스터 (GB200 NVL72 외부): **400 G / 800 G InfiniBand 또는 RoCE v2** + non-blocking fat-tree.
- 한 학습 job 의 all-reduce 트래픽이 클러스터 internal 대역폭의 80%+ 점유.

## 8. 함정

1. **ECMP 의 hash polarization** — 같은 hash 함수가 leaf / spine 양쪽에서 → 일부 path 만 사용. spine 마다 다른 hash seed 필요.
2. **MTU 미스매치** — 한 hop 만 1500 이면 9000 jumbo 가 fragment.
3. **L2 broadcast storm** — STP / RSTP 없이 loop 만들면 망 마비.
4. **VLAN 4094 한계** — 클라우드 멀티 테넌트는 VXLAN / Geneve 필요.
5. **광 단자 청결** — 먼지로 BER (Bit Error Rate) ↑.
6. **버퍼 부족 → microburst drop** — TCP 의 burst 트래픽 처리에 큰 buffer ASIC 권장.

## 9. 관련

- [[network-hardware]]
- [[rdma-infiniband]]
- [[../../network/network]] — IP / TCP / OSI
