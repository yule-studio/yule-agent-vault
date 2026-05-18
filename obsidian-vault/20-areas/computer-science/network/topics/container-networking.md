---
title: "container networking — network namespace / veth / bridge / overlay (VXLAN)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T14:30:00+09:00
tags: [network, topics, container-networking, namespace, veth, bridge, overlay, vxlan]
home_hub: network
related:
  - "[[topics]]"
  - "[[../network]]"
  - "[[../osi-7-layer/layer-2-data-link/switching-bridging]]"
  - "[[../osi-7-layer/layer-3-network/ip-routing-basics]]"
  - "[[../osi-7-layer/layer-3-network/fragmentation-mtu]]"
  - "[[../tools/iptables-netfilter]]"
  - "[[../../../devops/docker/docker-mental-models]]"
  - "[[../../../devops/kubernetes/kubernetes-mental-models]]"
  - "[[../../../devops/networking-ops/cni-deep]]"
  - "[[../../../devops/linux/linux-mental-models-for-devops]]"
---

# container networking — network namespace / veth / bridge / overlay (VXLAN)

**[[topics|↑ topics]]**

---

## 1. 목적

본 문서는 컨테이너 네트워킹을 **Linux 커널의 L2 / L3 primitive 의 합성** 으로 정의한다.

본 문서가 정의하는 것:
- network namespace 의 격리 모델
- veth pair 의 동작
- bridge / Linux bridge 의 L2 switching
- single-host 컨테이너 네트워크 (Docker bridge 의 실제 구성)
- multi-host overlay (VXLAN / GRE / WireGuard) 의 encapsulation
- k8s 의 4 가지 네트워킹 요구사항 (pod-pod / pod-node / pod-svc / 외부-svc)

본 문서가 정의하지 않는 것:
- 특정 CNI plugin 의 구현 (Calico / Cilium / Flannel) — [[../../../devops/networking-ops/cni-deep]]
- iptables 룰 syntax — [[../tools/iptables-netfilter]]
- container runtime / cgroup — [[../../../devops/docker/docker-mental-models]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 OS | Linux (커널 4.x 이상) |
| 대상 컨테이너 | Docker, containerd, CRI-O |
| 대상 오케스트레이션 | Kubernetes (CNI spec) |
| 제외 | Windows container networking — 별도 모델 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **network namespace (netns)** | network stack (인터페이스 / 라우팅 테이블 / 소켓 / iptables) 을 격리한 단위. |
| **veth pair** | 가상 이더넷 페어. 한쪽에 보낸 packet 이 다른쪽에서 나오는 양방향 pipe. |
| **bridge** | 같은 네트워크에 속한 인터페이스를 L2 로 연결하는 가상 스위치. |
| **Linux bridge** | 커널 내장 bridge 구현. `brctl` / `ip link add type bridge`. |
| **macvlan / ipvlan** | physical NIC 위에 가상 NIC 를 만들어 호스트 외부에 노출. |
| **overlay** | underlay (L3) 위에 가상 L2 네트워크를 만드는 캡슐화 (VXLAN / GRE / Geneve). |
| **VXLAN** | UDP 4789 위에 L2 frame 을 encapsulate 하는 overlay 표준. |
| **CNI (Container Network Interface)** | k8s / containerd 가 pod 생성 시 호출하는 네트워크 plugin spec. |
| **MTU** | 한 packet 의 최대 페이로드 크기. overlay 는 encapsulation overhead 만큼 줄여야 한다. |

---

## 4. network namespace — 격리 단위

### 4.1 정의

network namespace 는 다음 자원을 격리한다.

| 자원 | 효과 |
| --- | --- |
| 네트워크 인터페이스 (eth0 / lo) | 각 namespace 가 별도 lo 와 자기 인터페이스 보유 |
| 라우팅 테이블 | `ip route` 가 namespace 마다 다름 |
| iptables / nftables 룰 | namespace 별 chain |
| TCP/UDP 포트 공간 | 같은 포트를 서로 다른 namespace 가 동시 listen 가능 |
| ARP / neighbor 캐시 | 격리 |
| sysctl 의 net.* | 일부 namespace 별 (net.ipv4.ip_forward 등) |

### 4.2 관찰

```bash
# namespace 목록
ip netns list
lsns -t net

# 새 namespace 생성
ip netns add ns1
ip netns exec ns1 ip a              # 안에서 명령 실행

# process 의 net namespace 진입
nsenter -t <pid> -n ip a
nsenter -t <pid> -n ss -tuln

# 컨테이너의 net namespace
docker run -d --name web nginx
PID=$(docker inspect --format '{{.State.Pid}}' web)
nsenter -t $PID -n ip a
```

### 4.3 격리되지 않는 것

| 자원 | 비고 |
| --- | --- |
| host 의 NIC 드라이버 | 같은 NIC 를 여러 namespace 가 nail 할 수 없음 (별도 가상 인터페이스 필요) |
| Unix domain socket | namespace 와 무관 (filesystem path 기반) |
| netlink (일부) | namespace 별로 분리되지만 root 권한으로 cross |

---

## 5. veth pair — namespace 간 연결

### 5.1 정의

veth pair 는 두 인터페이스 (`vethA`, `vethB`) 로 구성된 가상 케이블이다. 한쪽에 송신된 packet 은 다른쪽에서 즉시 수신된다.

```
ns1 (vethA) ──┐
              ├──< 같은 veth pair >─
ns0 (vethB) ──┘
```

### 5.2 구성 예 (수동)

```bash
# veth pair 생성
ip link add veth_host type veth peer name veth_ns1

# 한쪽을 ns1 으로 이동
ip netns add ns1
ip link set veth_ns1 netns ns1

# 양쪽 인터페이스 활성화 + IP 할당
ip addr add 10.0.0.1/24 dev veth_host
ip link set veth_host up

ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth_ns1
ip netns exec ns1 ip link set veth_ns1 up
ip netns exec ns1 ip link set lo up

# 검증
ip netns exec ns1 ping 10.0.0.1
```

### 5.3 컨테이너 네트워킹의 기본 단위

| 컨테이너 | 안쪽 인터페이스 | 바깥쪽 인터페이스 |
| --- | --- | --- |
| Docker (default bridge) | `eth0` (컨테이너 안) | `vethXXXX` (host 의 `docker0` bridge 에 연결) |
| k8s (대부분 CNI) | `eth0` (pod 안) | `vethXXXX` (host 의 CNI bridge / TUN) |

---

## 6. Linux bridge — single-host L2 switch

### 6.1 정의

Linux bridge 는 같은 네트워크 segment 의 여러 인터페이스를 L2 로 연결하는 가상 스위치이다. MAC learning / flooding / STP 까지 지원.

### 6.2 Docker bridge network 의 실제 구성

```
┌────────────────────────────────────────────┐
│ host                                         │
│                                              │
│  ┌──────────┐    ┌──────────┐               │
│  │ ns:cA    │    │ ns:cB    │               │
│  │ eth0     │    │ eth0     │               │
│  │ 172.17.. │    │ 172.17.. │               │
│  └──┬───────┘    └──┬───────┘               │
│     │ veth pair       │ veth pair             │
│  ┌──┴───────┐    ┌──┴───────┐               │
│  │ vethA    │    │ vethB    │               │
│  └──┬───────┘    └──┬───────┘               │
│     │                 │                        │
│  ┌──┴─────────────────┴───────┐               │
│  │ docker0 (Linux bridge)     │ 172.17.0.1   │
│  └──────────┬─────────────────┘               │
│             │ iptables MASQUERADE              │
│  ┌──────────┴─────────┐                       │
│  │ eth0 (host NIC)    │ 192.168.1.10         │
│  └────────────────────┘                       │
└────────────────────────────────────────────┘
```

흐름:
1. 각 컨테이너의 net namespace 에 `eth0` (veth 한쪽)
2. veth 의 다른 쪽 (`vethA`, `vethB`) 은 host 의 `docker0` bridge 에 연결
3. 같은 bridge 의 컨테이너끼리는 L2 통신
4. 외부로 나갈 때 host 의 NAT (MASQUERADE) 로 source IP 변환
5. 외부에서 들어올 때는 `-p 80:8080` 의 DNAT 룰

### 6.3 구성 예 (수동 재현)

```bash
# bridge 생성
ip link add my0 type bridge
ip link set my0 up
ip addr add 10.10.0.1/24 dev my0

# 컨테이너 A 시뮬레이션
ip netns add cA
ip link add vethA type veth peer name vethA-br
ip link set vethA netns cA
ip link set vethA-br master my0           # bridge 에 연결
ip link set vethA-br up
ip netns exec cA ip addr add 10.10.0.2/24 dev vethA
ip netns exec cA ip link set vethA up
ip netns exec cA ip link set lo up
ip netns exec cA ip route add default via 10.10.0.1

# 외부로 나가도록 NAT
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -j MASQUERADE
```

### 6.4 alternative — macvlan / ipvlan

| 모드 | 효과 | 사용 |
| --- | --- | --- |
| **macvlan** | 컨테이너가 host NIC 의 같은 L2 에 직접 (자기 MAC) | LAN 의 별도 IP 가 필요한 컨테이너 (legacy DHCP / 외부 노출) |
| **ipvlan L2** | 컨테이너가 host 와 같은 MAC, 별도 IP | NIC 의 MAC 한도 / promisc 모드 회피 |
| **ipvlan L3** | 컨테이너가 별도 routed subnet | overlay 없이 routable pod IP |

---

## 7. overlay — multi-host 네트워크

### 7.1 문제

여러 host 의 컨테이너가 같은 가상 L2 네트워크처럼 보이도록 하려면 host 간 underlay (L3 IP) 위에 L2 frame 을 캡슐화해야 한다.

### 7.2 VXLAN

```
[node A]  pod A ── vethA ── cni-bridge ──┐
                                          │ encap (VTEP)
                                          ▼
                                    UDP 4789 (VXLAN)
                                          │
                                    [physical IP network]
                                          │
                                          ▼
[node B]  cni-bridge ── vethB ── pod B  decap (VTEP)
```

| 항목 | 값 |
| --- | --- |
| 전송 protocol | UDP 4789 |
| header | 8 byte VXLAN + 8 byte UDP + 20 byte IP + 14 byte Ethernet = 50 byte overhead |
| VNI (Virtual Network ID) | 24-bit (tenant 격리) |
| 일반 MTU | underlay 1500 → overlay 1450 권장 |

### 7.3 다른 overlay 방식

| 방식 | 캡슐화 | overhead | 사용 |
| --- | --- | --- | --- |
| VXLAN | UDP 4789 | 50 byte | 표준 (Flannel / Calico VXLAN / Cilium VXLAN) |
| Geneve | UDP 6081 | 50+ byte (가변) | Open vSwitch / OVN |
| GRE | IP 47 | 24 byte | legacy / cisco |
| IPsec | ESP 50 / AH 51 | 50+ byte | encrypted overlay (Calico IPsec) |
| WireGuard | UDP custom | 32 byte | encrypted overlay (Calico / Cilium WG) |
| native routing | (캡슐화 없음) | 0 | BGP 로 pod CIDR advertise (Calico BGP / Cilium native) |

### 7.4 underlay vs overlay

| 모델 | 장점 | 단점 |
| --- | --- | --- |
| overlay (VXLAN) | underlay 무관 / pod IP 충돌 없음 | encap overhead / MTU 신경써야 |
| native routing (BGP) | overhead 없음 / 성능 ↑ | underlay 가 BGP 지원 필요 / pod CIDR routable |

---

## 8. k8s 의 4 네트워킹 요구사항

CNI 가 충족해야 하는 4 가지:

| 요구사항 | 의미 |
| --- | --- |
| pod-pod (같은 node) | NAT 없이 직접 통신 |
| pod-pod (다른 node) | NAT 없이 직접 통신 |
| pod-node | NAT 없이 양방향 |
| 외부-svc | NodePort / LoadBalancer / Ingress 경유 |

이를 위해 CNI 는 다음 중 하나 (혹은 결합) 선택:

| 방식 | 예 |
| --- | --- |
| bridge + overlay (VXLAN) | Flannel VXLAN, Calico VXLAN |
| bridge + BGP (native routing) | Calico BGP, Cilium native routing |
| eBPF + native | Cilium eBPF |
| AWS VPC CNI | pod IP = ENI secondary IP (overlay 없음) |

상세: [[../../../devops/networking-ops/cni-deep]].

---

## 9. service / kube-proxy 와의 결합

pod 간 통신 위에 Service 가 얹힌다.

| 컴포넌트 | 역할 |
| --- | --- |
| Service | virtual IP (ClusterIP) + port |
| kube-proxy (iptables) | ClusterIP 로 들어온 packet 을 random pod IP 로 DNAT |
| kube-proxy (IPVS) | ClusterIP 를 IPVS virtual server 로 등록 |
| kube-proxy (Cilium eBPF) | eBPF map 으로 svc → pod 직접 forward |

Service 의 packet flow (iptables mode):

```
pod A → ClusterIP 10.96.0.5:80
   ↓ (PREROUTING iptables KUBE-SERVICES → KUBE-SVC-XXX)
   ↓ (DNAT to pod B 10.244.1.7:8080)
pod B
```

상세: [[../tools/iptables-netfilter]] §8, [[../../../devops/kubernetes/kubernetes-mental-models]].

---

## 10. MTU 와 fragmentation

### 10.1 문제

overlay 는 encapsulation overhead 만큼 inner MTU 를 줄여야 한다. 잘못 설정 시 packet drop / 응답 hang.

### 10.2 권장 값

| 환경 | underlay MTU | overlay (VXLAN) MTU |
| --- | --- | --- |
| 일반 ethernet | 1500 | 1450 |
| jumbo frame | 9000 | 8950 |
| AWS EKS (overlay 없음) | 9001 | 9001 |
| AWS VPC peering | 1500 | (적용 X) |

### 10.3 검증

```bash
ip link show eth0          # MTU 확인
ping -M do -s 1472 8.8.8.8 # don't fragment + 1472 byte payload (1500 MTU 가정)
tracepath 8.8.8.8          # Path MTU 측정
```

상세: [[../osi-7-layer/layer-3-network/fragmentation-mtu]].

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 모델 설명 |
| --- | --- | --- |
| pod 간 통신은 되는데 응답이 느림 | overlay MTU 미설정 → fragmentation | §10 |
| 같은 node 의 pod 간 통신 됨, 다른 node 안 됨 | underlay 가 overlay port (UDP 4789) 차단 | §7.2 |
| pod 가 외부 접근 못함 | host 의 `ip_forward = 0` 또는 MASQUERADE 룰 부재 | §6.2 |
| Service IP 가 응답 없음 | kube-proxy 죽음 또는 iptables 룰 부재 | §9, [[../tools/iptables-netfilter]] |
| pod IP 가 외부에서 직접 도달 | AWS VPC CNI 또는 BGP routing — 의도면 OK, 아니면 NetworkPolicy 부재 | §7.4 |
| 같은 LAN 의 컨테이너 두 host 가 IP 충돌 | overlay 의 VNI 격리 미적용 또는 같은 subnet | §7.2 |
| macvlan 컨테이너가 host 통신 불가 | macvlan 의 동작 — host 와는 직접 못 함 (별도 macvlan 인터페이스 필요) | §6.4 |
| WireGuard overlay 후 throughput 감소 | encryption + UDP encap overhead | §7.3 |

---

## 12. 참고

- [[topics|↑ topics]]
- [[../network|↑ network hub]]
- [[../osi-7-layer/layer-2-data-link/switching-bridging]] — L2 bridge
- [[../osi-7-layer/layer-3-network/ip-routing-basics]] — L3 routing
- [[../osi-7-layer/layer-3-network/fragmentation-mtu]] — MTU / fragmentation
- [[../osi-7-layer/layer-3-network/ipsec-vpn]] — IPsec encapsulation
- [[../ssh-rdp-vpn/wireguard]] — WireGuard tunneling
- [[../tools/iptables-netfilter]] — kube-proxy / Docker bridge 의 packet 흐름
- [[../../../devops/docker/docker-mental-models]] — container = process + namespace + cgroup
- [[../../../devops/kubernetes/kubernetes-mental-models]] — Service / kube-proxy
- [[../../../devops/networking-ops/cni-deep]] — CNI plugin 별 구현
- [[../../../devops/linux/linux-mental-models-for-devops]] — namespace / cgroup
- CNI spec — https://github.com/containernetworking/cni
