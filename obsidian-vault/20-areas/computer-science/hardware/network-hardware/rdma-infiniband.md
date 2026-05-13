---
title: "RDMA·InfiniBand"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, rdma, roce, iwarp, infiniband, ucx, gpu-direct]
---

# RDMA·InfiniBand

**[[network-hardware|↑ 네트워크 하드웨어]]**

## 1. RDMA — Remote Direct Memory Access

원격 호스트의 메모리 ↔ 로컬 메모리를 **CPU / OS 를 거치지 않고** NIC 가 직접 옮김.

### 흐름
1. 로컬 응용이 NIC 에 "원격 호스트 A 의 메모리 region X 에서 1 MB 를 내 메모리 Y 로 옮겨" 명령 (verbs API).
2. 로컬 NIC 가 원격 NIC 에 RDMA Read 패킷 전송.
3. 원격 NIC 가 원격 메모리에서 직접 읽어 응답.
4. 로컬 NIC 가 응답을 로컬 메모리 Y 에 직접 쓰기.
5. 완료 시 응용에 completion event.

### 장점
- **OS / CPU 거의 0% 사용** — 응용은 응용 일에 집중.
- **latency 1-2 μs** — TCP 의 50-100 μs 대비 50× 빠름.
- **throughput** — 200/400/800 Gbps NIC 의 line rate.

### 단점
- 응용이 RDMA verbs API 로 작성되어야 함.
- 메모리 region 사전 등록 + 권한 키 교환.
- 네트워크가 lossless 거의 필수 (drop 시 latency 폭증).

## 2. 전송 방식 3 종

| 전송 | 위에 | 특징 |
| --- | --- | --- |
| **RoCE v1** | Ethernet (L2) | 같은 subnet 내. 거의 미사용. |
| **RoCE v2** | UDP/IP (L3) | 라우팅 가능. 현재 데이터센터 표준. |
| **iWARP** | TCP | TCP 위 RDMA. PFC 없어도 동작. latency 약간 ↑. |
| **InfiniBand (native)** | InfiniBand fabric | HPC / AI 전용 fabric. lossless. |

## 3. RoCE v2 lossless 조건

RoCE 는 packet drop 시 latency 폭증. lossless 네트워크 필수:

- **PFC (Priority Flow Control, 802.1Qbb)** — 한 priority 가 혼잡하면 send 쪽에 "잠시 멈춰" pause 프레임.
- **ECN (Explicit Congestion Notification)** — 혼잡 표시. RoCE 의 DC-QCN 알고리즘이 ECN 으로 속도 조절.
- 위 둘이 모두 enable + 튜닝 되어야 production RoCE.

## 4. InfiniBand

NVIDIA (구 Mellanox) 가 주도하는 전용 fabric.

| 세대 | per-lane | 4× / link |
| --- | --- | --- |
| SDR | 2.5 G | 10 G |
| DDR | 5 G | 20 G |
| QDR | 10 G | 40 G |
| FDR | 14 G | 56 G |
| EDR | 25 G | 100 G |
| HDR | 50 G | 200 G |
| NDR | 100 G | 400 G |
| XDR | 200 G | 800 G (예정) |

### 장점
- **Lossless by design** — credit-based flow control 이 처음부터.
- **Adaptive routing** — 혼잡 회피.
- **SHARP** — switch ASIC 가 all-reduce / broadcast 등 collective 직접 처리.

### 단점
- 전용 NIC + 스위치 (NVIDIA Quantum). 비싸다.
- Ethernet vs IB 양자 택일 (또는 둘 다).

## 5. NVMe-oF over RDMA

- NVMe 명령을 RDMA 위로.
- 분리형 스토리지 (storage chassis 에 NVMe, compute 노드는 NIC 만).
- **CPU 0% 로 원격 SSD 직접 read/write**.

## 6. GPUDirect / Storage / RDMA

- **GPUDirect RDMA** — 다른 호스트 GPU 메모리 ↔ 로컬 GPU 메모리 직접. CPU / 호스트 메모리 우회.
- **GPUDirect Storage** — NVMe ↔ GPU 메모리 직접.
- AI 학습 (DGX / HGX) 에서 all-reduce 가속의 핵심.

## 7. UCX / MPI

- **UCX (Unified Communication X)** — RDMA / shared memory / TCP 위 추상화. OpenMPI, NCCL 의 backend.
- **NCCL** — NVIDIA GPU 전용 collective communication.
- **RCCL** — AMD 대응.

## 8. 진단

```bash
# RDMA 장치 목록
ibv_devices
ibstat
mlxlink -d /dev/mst/mt4123_pciconf0

# 성능 측정
ib_write_bw  / ib_read_bw  / ib_send_bw
ib_write_lat / ib_read_lat / ib_send_lat

# PFC / ECN 확인 (RoCE)
mlnx_qos -i eth0
sudo dcb pfc show dev eth0
```

## 9. 클라우드의 RDMA

- **AWS EFA (Elastic Fabric Adapter)** — UDP 위 SRD (Scalable Reliable Datagram). RDMA-like.
- **Azure InfiniBand HBv* / NDv4** — native IB.
- **GCP** — RoCE 일부 인스턴스.

## 10. 함정

1. **PFC / ECN 미튜닝 → packet drop → latency 폭증** — RoCE production 의 가장 흔한 실수.
2. **memory region 등록 부담** — 자주 region pin/unpin 하면 RDMA latency 가 TCP 보다 느려질 수 있음. memory pool 추천.
3. **firewall 가 UDP/4791 (RoCE v2) 차단** — VPC 정책 확인.
4. **GPUDirect 의 IOMMU 호환** — 일부 보드는 IOMMU = on 시 GPUDirect 성능 ↓ — passthrough mode (iommu=pt) 필요.
5. **InfiniBand 와 Ethernet 혼동** — 같은 QSFP-DD 포트지만 fabric 다름. switch 와 NIC 모드 일치 필수.

## 11. 관련

- [[network-hardware]]
- [[nic-offload]]
- [[switch-fabric]]
- [[../bus-io/pcie]] — RDMA NIC 는 PCIe Gen4/5 x16
- [[../soc/datacenter-accelerators]] — GPUDirect, NVLink, NCCL
