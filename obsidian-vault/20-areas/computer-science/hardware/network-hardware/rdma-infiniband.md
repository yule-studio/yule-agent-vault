---
title: "RDMA·InfiniBand"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, rdma, roce, iwarp, infiniband, verbs, qp, cq, mr, pfc, ecn, dcqcn, gpu-direct, ucx, nccl]
---

# RDMA·InfiniBand

**[[network-hardware|↑ 네트워크 하드웨어]]**

> 원격 호스트 메모리 ↔ 로컬 메모리를 OS / CPU 를 거의 거치지 않고 옮기는 기술. AI 학습 / HPC / 고성능 분산 시스템의 표준 fabric.

## 1. RDMA 의 본질

```
[전통 TCP/IP]
   Sender App
     ↓
   syscall (copy)
     ↓
   Kernel TCP stack (copy)
     ↓
   NIC DMA
     ↓ ── 네트워크 ──↓
   NIC DMA
     ↓
   Kernel TCP stack (copy)
     ↓
   syscall (copy)
     ↓
   Receiver App

→ 4 copies, 2 syscall, OS bottleneck
```

```
[RDMA]
   Sender App
     ↓ (post WR, register MR)
   NIC ── direct DMA from app memory ── 네트워크 ── 직접 receiver memory
     ↓ (completion event)
   Receiver App

→ 0 copy, 0 syscall (after setup), ns latency
```

## 2. 핵심 매개변수

| 메트릭 | TCP/IP (10 GbE) | RDMA (RoCE v2, 100 GbE) |
| --- | --- | --- |
| latency (1-side ping) | 50-100 μs | 1-2 μs |
| CPU overhead | 1 core 100% @ line rate | < 5% |
| throughput | 8-9 Gbps (overhead) | line rate (~95 Gbps) |
| 메모리 copy | 4 회 | 0 회 |

## 3. RDMA 의 3 가지 작업 모드

### One-sided (Read / Write)
- **상대의 CPU 가 관여하지 않음**.
- A 가 B 의 메모리 region X 에서 직접 read 또는 write.
- B 는 사전에 region 등록 + remote key 만 공유.

### Two-sided (Send / Recv)
- 전통 메시징 비슷. 송신자 send, 수신자 recv.
- 양쪽 CPU 가 일부 관여 (post WR + completion handling).

### Atomic (Compare-and-Swap, Fetch-and-Add)
- 원격 메모리에 atomic operation.
- 분산 lock / shared counter / lock-free 자료구조.

## 4. RDMA Verbs API — 핵심 객체

```c
#include <infiniband/verbs.h>
```

### 객체 모델

| 객체 | 의미 |
| --- | --- |
| **Device** | RDMA-capable NIC (e.g. ConnectX) |
| **Context** | NIC 와의 connection. ibv_open_device |
| **PD (Protection Domain)** | 보안 격리 단위. 같은 PD 안에서만 MR/QP 공유. |
| **MR (Memory Region)** | RDMA 가 접근 가능한 메모리. pin + DMA mapping + key 발급. |
| **QP (Queue Pair)** | Send Queue + Receive Queue. RDMA 통신의 endpoint. |
| **CQ (Completion Queue)** | WR 완료 이벤트 받음. |
| **WR (Work Request)** | "이 메모리에서 저 메모리로 write 해" 같은 명령. |
| **WC (Work Completion)** | WR 결과. CQ 에서 polling 또는 event-driven. |

### 흐름

```
1. ibv_open_device(dev)
2. ibv_alloc_pd(ctx)
3. ibv_reg_mr(pd, addr, len, ACCESS_FLAGS)  ← 메모리 pin + DMA map
4. ibv_create_cq(ctx, depth)
5. ibv_create_qp(pd, qp_attr)
6. ibv_modify_qp(qp, INIT → RTR → RTS)      ← 양쪽 QP 정보 교환 후 connect
7. ibv_post_send(qp, wr) / ibv_post_recv(qp, wr)
8. ibv_poll_cq(cq, &wc)                     ← 완료 확인
```

### Memory Region 의 권한

| Flag | 의미 |
| --- | --- |
| `IBV_ACCESS_LOCAL_WRITE` | 로컬 NIC 가 메모리에 write |
| `IBV_ACCESS_REMOTE_READ` | 원격 호스트가 read |
| `IBV_ACCESS_REMOTE_WRITE` | 원격 호스트가 write |
| `IBV_ACCESS_REMOTE_ATOMIC` | 원격 atomic 작업 |

→ remote 권한이 있으면 그 region 의 **rkey (remote key)** 가 발급되고 원격 호스트에 공유 필요.

## 5. QP 종류 (Transport Service Type)

| QP type | 의미 | 신뢰성 | 순서 | 대표 사용 |
| --- | --- | --- | --- | --- |
| **RC (Reliable Connected)** | 신뢰성 있는 1:1 connection | ✓ | ✓ | 일반 RDMA, 가장 흔함 |
| **UC (Unreliable Connected)** | 신뢰 없는 1:1 connection | ✗ | ✓ | RDMA Write 만 |
| **UD (Unreliable Datagram)** | UDP 비슷, 1:N | ✗ | ✗ | multicast, name service |
| **RD (Reliable Datagram)** | 신뢰 있는 1:N | ✓ | ✗ | 거의 미사용 |
| **XRC (eXtended RC)** | RC 의 확장. 큰 클러스터 scalable. | ✓ | ✓ | HPC 슈퍼컴 |
| **DC (Dynamically Connected, MLX 전용)** | RC 가 동적 연결 | ✓ | ✓ | NVIDIA/Mellanox |

### QP 상태 머신

```
RESET ─→ INIT ─→ RTR (Ready To Receive) ─→ RTS (Ready To Send) ⇄ SQE/ERR
                                              │
                                              ▼
                                          NORMAL OPERATION
```

연결 setup:
1. 양쪽 RESET.
2. modify_qp 로 INIT.
3. 상대 QPN / GID / 초기 sequence number 교환 (OOB — socket, file 등).
4. modify_qp 로 RTR (recv 받을 준비).
5. modify_qp 로 RTS (send 가능).

## 6. 전송 방식 — RoCE / iWARP / native IB

| 전송 | 위에 | lossless 요구 | 특징 |
| --- | --- | --- | --- |
| **RoCE v1** | Ethernet (L2) | ✓ | 같은 subnet 내. 거의 미사용. |
| **RoCE v2** | UDP/IP (L3) | ✓ | 라우팅 가능. 현재 데이터센터 표준. |
| **iWARP** | TCP | ✗ (TCP 가 알아서) | TCP 위 RDMA. PFC 없어도 동작. 약간 ↑ latency. |
| **InfiniBand (native)** | InfiniBand fabric | ✓ (credit-based) | HPC / AI 전용 fabric. |

## 7. RoCE v2 — Ethernet 위 RDMA 의 lossless 조건

RoCE 의 약점: packet drop 시 latency 폭증 (timeout + 재전송).

→ lossless 네트워크 필수:

### PFC (Priority Flow Control, 802.1Qbb)
- 한 priority 가 혼잡하면 송신 쪽에 "잠시 멈춰" pause 프레임.
- 802.3x global PAUSE 의 priority 별 버전.
- RoCE 가 가장 의존하는 메커니즘.

### ECN (Explicit Congestion Notification)
- 혼잡한 라우터가 IP 헤더에 표시.
- 수신측이 송신측에 알림.
- RoCE 의 **DC-QCN (Data Center QCN)** 알고리즘이 ECN 으로 속도 자동 조절.

### DC-QCN 동작
1. switch 가 queue 가 임계치 (예: 50%) 넘으면 ECN-marked packet 송신.
2. 수신측이 송신측에 **CNP (Congestion Notification Packet)** 전송.
3. 송신측이 송신율 감소 (BIC / cubic 등 알고리즘).
4. 일정 시간 후 다시 증가.

→ TCP cubic 의 RDMA 버전. switch 와 NIC 양쪽 튜닝 필요.

### 진단

```bash
# PFC 상태
sudo dcb pfc show dev eth0
sudo mlnx_qos -i eth0

# ECN 통계
sudo ethtool -S eth0 | grep -E 'rx_pause|tx_pause|ecn'

# RoCE 통계
cat /sys/class/infiniband/mlx5_0/ports/1/hw_counters/* | head -20
```

## 8. InfiniBand — 전용 lossless fabric

NVIDIA (구 Mellanox) 가 주도.

### 세대

| 세대 | per-lane (Gb/s) | 4× (Gb/s) | 12× |
| --- | --- | --- | --- |
| SDR | 2.5 | 10 | 30 |
| DDR | 5 | 20 | 60 |
| QDR | 10 | 40 | 120 |
| FDR | 14 | 56 | 168 |
| EDR | 25 | 100 | 300 |
| HDR | 50 | 200 | 600 |
| **NDR** | 100 | 400 | 1200 |
| **XDR** (예정) | 200 | 800 | 2400 |

### Credit-based Flow Control
- credit 0 면 송신 금지 → 본질적 lossless.
- end-to-end 보장.

### Adaptive routing
- 혼잡 path 회피해서 dynamic 선택.
- packet 순서 깨질 수 있어 RC 의 reorder buffer 필수.

### SHARP (Scalable Hierarchical Aggregation and Reduction Protocol)
- switch ASIC 가 all-reduce / broadcast / barrier 같은 collective 직접 처리.
- 일반 host-driven all-reduce 대비 2-3 배 빠름.
- AI 학습의 핵심 가속.

### IB 단점
- 전용 NIC + 스위치 (NVIDIA Quantum / Quantum-2). 비쌈.
- Ethernet 과 양립 (한 클러스터 안 IB + Ethernet 동시 운영).

## 9. NVMe-oF over RDMA

NVMe 명령을 RDMA 위로.

```
   Compute Node                Storage Node
     │                            │
   Application                  NVMe SSD
     │                            │
   NVMe Driver                   NVMe Target
     │                            │
   NVMe-oF Init                  NVMe-oF Tgt
     │                            │
   RDMA NIC ── RoCE/IB/iWARP ──── RDMA NIC
```

- **CPU 0% 로 원격 SSD 직접 read/write**.
- 분리형 스토리지 (composable infrastructure).
- 데이터센터 SSD pool.
- AI 학습 dataset 공유 read.

```bash
# Linux nvme-cli
sudo nvme discover -t rdma -a 10.0.0.1 -s 4420
sudo nvme connect -t rdma -n nqn.2024-01.com.example:disk1 -a 10.0.0.1 -s 4420
sudo nvme list
```

## 10. GPUDirect — GPU 메모리 직접

### GPUDirect RDMA
- 다른 호스트 GPU 메모리 ↔ 로컬 GPU 메모리 직접.
- 호스트 메모리 / CPU 우회.
- AI 학습 (DGX / HGX) 의 all-reduce 가속 핵심.

### GPUDirect Storage (GDS)
- NVMe ↔ GPU 메모리 직접.
- 호스트 RAM 거치지 않음.
- AI dataset 의 GPU loading 가속.

### GPUDirect P2P (같은 노드 안 GPU 간)
- 한 노드 안 GPU 끼리 PCIe / NVLink 로 직접.
- CUDA P2P API.

### 사전 조건
- IOMMU 활성 (또는 `iommu=pt`).
- NVIDIA Mellanox NIC + 같은 PCIe root complex / NVSwitch.
- 호스트 OS 의 nvidia-peermem 모듈.

## 11. UCX / NCCL / RCCL

### UCX (Unified Communication X)
- RDMA / shared memory / TCP 위 추상화 라이브러리.
- OpenMPI, MPICH 의 backend.
- 어떤 fabric 이든 같은 API.

### NCCL (NVIDIA Collective Communications Library)
- NVIDIA GPU 전용 collective.
- AllReduce, AllGather, Broadcast, ReduceScatter.
- NVLink / PCIe / RDMA (RoCE / IB) 자동 활용.
- 토폴로지 자동 검출 (NVSwitch / NVLink topology aware).

### RCCL
- AMD GPU 의 대응. ROCm Communication Collectives Library.

### MPI 표준
- 옛 HPC 의 표준. RDMA 위 collective.
- 현재 AI 영역에서는 NCCL/RCCL 이 dominant.

## 12. Linux RDMA 진단

```bash
# RDMA 장치
sudo apt install rdma-core ibverbs-utils perftest
ibv_devices                # 장치 목록
ibstat                     # status, 포트, GID
ibv_devinfo                # 자세히
sudo show_gids             # GID table

# 케이블 / 광 / link health (Mellanox)
sudo mlxlink -d /dev/mst/mt4123_pciconf0 --show_eye --show_errors

# 성능 측정 (양쪽 노드에서)
# A 쪽 (server)
ib_write_bw  -d mlx5_0
# B 쪽 (client)
ib_write_bw  -d mlx5_0 <A_IP>

# 다양한 transport
ib_write_lat
ib_write_bw
ib_read_lat
ib_read_bw
ib_send_lat
ib_send_bw
ib_atomic_bw

# PFC / ECN / 통계
cat /sys/class/infiniband/mlx5_0/ports/1/hw_counters/* | head
sudo dcb pfc show dev eth0
sudo ethtool -S eth0 | grep -iE 'pfc|prio_pause'
```

## 13. 튜닝 — 실 클러스터

### Buffer / queue 크기
- RDMA NIC 의 receive buffer 큐 크기.
- 너무 작으면 backpressure → PFC 폭주.
- 너무 크면 head-of-line blocking.

### MTU
- IB = 256 / 512 / 1024 / 2048 / 4096 byte (MTU).
- RoCE = Ethernet jumbo frame 9000.
- 큰 MTU = throughput ↑ + per-packet 오버헤드 ↓.

### IRQ affinity
- RDMA NIC 의 인터럽트를 NIC 와 같은 NUMA 노드의 코어로.

```bash
sudo mlnx_tune -p HIGH_THROUGHPUT
```

### NIC 의 cache mode
- DDIO (Data Direct I/O, Intel) — NIC 가 DMA 한 데이터가 LLC 에 즉시.
- AMD Smart Access — 비슷한 효과.

## 14. 클라우드의 RDMA

### AWS EFA (Elastic Fabric Adapter)
- UDP 위 SRD (Scalable Reliable Datagram).
- 진정한 RDMA 는 아니지만 RDMA-like 성능.
- HPC / ML 인스턴스 (P4d, P5, hpc7g).

### Azure InfiniBand (HBv*, NDv4, NDv5)
- native IB. HDR 200 Gbps / NDR 400 Gbps.
- HPC / 대규모 ML 학습.

### GCP
- A3 인스턴스에 RoCE.
- TPU pod 은 자체 inter-chip interconnect.

### Oracle Cloud (OCI)
- RoCE v2 클러스터 (BM.GPU4 등).

## 15. 한 ML 학습 워크로드의 RDMA flow

GPT 류 LLM 학습 (모델 70B, 32 노드, 256 GPU 가정):

1. **Forward pass**: 각 GPU 가 자기 layer 의 activation 계산.
2. **Backward pass**: gradient 계산.
3. **AllReduce gradient** (NCCL):
   - NCCL 가 토폴로지 보고 ring 또는 tree 알고리즘 선택.
   - 같은 노드 안: NVLink 또는 PCIe P2P.
   - 노드 사이: RoCE v2 또는 IB 의 GPUDirect RDMA.
   - 한 step 의 AllReduce 가 모델 크기 / 노드 수 의 함수.
4. **Optimizer step**: gradient 적용.

→ AllReduce 의 throughput · latency 가 step time 의 30~70%.
→ **클러스터 internal RDMA fabric 이 학습 throughput 의 천장**.

## 16. 함정

1. **PFC / ECN 미튜닝 → packet drop → latency 폭증** — RoCE production 의 가장 흔한 실수.
2. **memory region 등록 부담** — 자주 region pin/unpin 하면 RDMA latency 가 TCP 보다 느려질 수도. memory pool 추천.
3. **PFC storm** — 한 호스트의 NIC 가 끊임없이 PAUSE 보내 인접 호스트도 멈춤. priority 별 분리 필수.
4. **firewall 가 UDP/4791 (RoCE v2) 차단** — VPC 정책 확인.
5. **GPUDirect 의 IOMMU 호환** — 일부 보드는 IOMMU = on 시 GPUDirect 성능 ↓ — passthrough mode (`iommu=pt`) 필요.
6. **InfiniBand 와 Ethernet 혼동** — 같은 QSFP-DD 포트지만 fabric 다름. switch 와 NIC 모드 일치 필수.
7. **QP 수 폭주** — 큰 클러스터 (1000+ 노드) 의 N×N pairwise QP = NIC 메모리 한계. XRC 또는 DC 사용.
8. **RoCE 의 ECMP hash polarization** — leaf-spine 의 multipath 가 hash 함수가 같으면 일부 path 만 사용.
9. **lossy network 위 RoCE** — drop 1% 면 throughput 90%+ 손실.
10. **MTU mismatch** — 9000 jumbo 가 일부 hop 에서 1500 으로 fragment → throughput 폭락.
11. **NCCL P2P fallback to host memory** — 의도 안 한 fabric 으로 떨어짐. `NCCL_DEBUG=INFO` 로 확인.
12. **rkey 관리** — region 등록 후 rkey 를 양쪽에 안전하게 교환 필요 (보안 + 일관성).

## 17. 관련

- [[network-hardware]]
- [[nic-offload]] — NIC HW 가속 일반
- [[switch-fabric]] — lossless fabric topology
- [[../bus-io/pcie]] — RDMA NIC = PCIe Gen4/5 x16
- [[../bus-io/dma-iommu]] — IOMMU + PASID + ATS
- [[../soc/datacenter-accelerators]] — H100/B100, NVLink, NVSwitch, NCCL
- [[../../distributed-systems/distributed-systems]] — 분산 시스템에서의 RDMA 활용
