---
title: "NIC 가속 기능 (Offload)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, nic, rss, tso, gro, gso, lro, checksum, dpdk, ebpf, xdp]
---

# NIC 가속 기능 (Offload)

**[[network-hardware|↑ 네트워크 하드웨어]]**

## 1. 왜 offload 인가

- 10 GbE = 14.88 Mpps (64-byte 패킷). 단일 CPU 코어로 처리하면 100% 점유.
- NIC 가 IP / TCP / UDP / TLS 일부를 직접 처리하면 CPU 가 응용에만 집중.

## 2. 핵심 Offload 표

| 기능 | 의미 | 효과 |
| --- | --- | --- |
| **Checksum offload** | IP/TCP/UDP 체크섬 계산 | CPU 0% 비용으로 검증 |
| **TSO (TCP Segmentation Offload)** | 큰 segment 를 NIC 가 잘라 보냄 | tx 코어 부담 ↓ |
| **GSO (Generic Segmentation Offload)** | OS 측 일반 segmentation | TSO 가 없는 NIC 에서도 효과 |
| **LRO (Large Receive Offload)** | 작은 패킷 묶어서 큰 segment 로 stack 에 전달 | rx 코어 부담 ↓ |
| **GRO (Generic Receive Offload)** | OS 측 묶음 (LRO 의 software 버전) | 가상화 친화 (LRO 는 forwarding 문제 가능) |
| **VLAN tagging / stripping** | NIC 가 VLAN 헤더 처리 | |
| **TLS offload** | 일부 NIC 가 TLS 1.2/1.3 record 직접 | Netflix CDN, Cloudflare 사용 |
| **IPsec offload** | crypto + 인증 NIC 처리 | |
| **VXLAN / Geneve encap offload** | 오버레이 헤더 처리 | 클라우드 가상 네트워크 |

## 3. RSS / RPS / RFS / aRFS

### RSS (Receive Side Scaling)
- 들어오는 패킷의 5-tuple (src IP / dst IP / src port / dst port / proto) 해시.
- 해시 → 큐 인덱스 → CPU 코어.
- 같은 flow 는 같은 큐 → cache 친화.
- 큐 수 = 보통 nproc 또는 그 절반.

### RPS (Receive Packet Steering)
- RSS 의 소프트웨어 버전. NIC 가 RSS 미지원 시.

### RFS / aRFS (Accelerated RFS)
- flow 가 어느 코어에서 처리되는지 추적해서 다음 패킷을 그 코어로.
- aRFS 는 NIC 의 ntuple filter 사용.

```bash
sudo ethtool -l eth0          # 큐 수
sudo ethtool -L eth0 combined 16
sudo ethtool -x eth0          # RSS hash 매핑
echo 'ff' | sudo tee /sys/class/net/eth0/queues/rx-0/rps_cpus
```

## 4. NAPI

- 1 패킷 = 1 인터럽트 → 14.88 Mpps × IRQ → CPU 폭주.
- NAPI = 인터럽트 + polling 의 hybrid. 자세히 [[../bus-io/dma-iommu]] §4.

## 5. DPDK (Data Plane Development Kit)

- 사용자 공간에서 NIC 를 직접 다루는 framework.
- 커널 우회 (kernel bypass).
- huge page + DMA 메모리 + busy-loop polling 으로 ns 단위 latency.
- 통신사 / CDN / 보안 어플라이언스에서 사용.
- 단점: 한 코어 100% busy-loop, 호환성 / 운영 복잡.

## 6. XDP (eXpress Data Path) + eBPF

- 커널 안에서 NIC 드라이버 받기 직전에 eBPF 프로그램 실행.
- 패킷 drop / forward / redirect / pass 결정.
- DPDK 의 일부 사용 사례를 커널 안에서 처리.
- Cilium, Cloudflare Magic Firewall 활용.

## 7. AF_XDP

- XDP 가 패킷을 사용자 공간으로 zero-copy 전달.
- DPDK 보다 커널 통합이 좋고, XDP 보다 사용자 공간 자유.
- 2018 부터 Linux 메인라인.

## 8. Time Stamping / PTP

- **Hardware Timestamping** — NIC 가 패킷 도착 / 출발 시각을 ns 정밀도로.
- **PTP (Precision Time Protocol, IEEE 1588)** — 네트워크 시계 동기. PPP < 1 μs.
- **NTP** = ms 정밀도. 금융 / 5G / 데이터센터는 PTP 필수.

## 9. NIC 진단

```bash
ethtool -i eth0                    # 드라이버 / firmware
ethtool -k eth0                    # offload 기능 on/off
ethtool -K eth0 gro on tso on      # 켜기
ethtool -S eth0                    # 통계 (drops, csum errors, queue별)
ethtool -L eth0 combined 16        # 큐 수
ethtool -c eth0                    # interrupt coalescing
ethtool -C eth0 rx-usecs 50

ip -s link
cat /proc/interrupts | grep mlx
```

## 10. 함정

1. **RSS 미사용** — 100 GbE 가 단일 코어 100% 로 막힘.
2. **GRO/LRO 의 forwarding 문제** — router / bridge 역할에서는 LRO 가 packet 변형으로 문제. forwarding 서버는 LRO off / GRO 만.
3. **TLS offload 의 키 노출** — 일부 구현은 키가 NIC 메모리에 . 보안 요구 검토.
4. **DPDK 와 일반 NIC 드라이버 혼용** — DPDK 가 NIC 를 점유하면 시스템 ping 도 불가.
5. **interrupt coalescing 너무 공격적** — latency 민감 응용에 100 μs 추가.
6. **MTU 9000 + offload** — 일부 NIC 의 TSO/GRO 가 jumbo 에서 버그.

## 11. 관련

- [[network-hardware]]
- [[rdma-infiniband]]
- [[../bus-io/dma-iommu]] — MSI-X / NAPI / IOMMU / SR-IOV
- [[../../network/network]] — 프로토콜 레이어
