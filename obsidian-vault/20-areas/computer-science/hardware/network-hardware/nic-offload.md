---
title: "NIC 가속 기능 (Offload)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, nic, rss, tso, gso, gro, lro, checksum, dpdk, ebpf, xdp, af_xdp, ptp, tls-offload]
---

# NIC 가속 기능 (Offload)

**[[network-hardware|↑ 네트워크 하드웨어]]**

> 10 GbE 이상에서 호스트 CPU 가 네트워크 처리에 잠식되지 않게 NIC 가 직접 처리하는 작업들. 현대 데이터센터 / 클라우드 의 핵심.

## 1. 왜 offload 인가

### 패킷 수 계산
- 1 GbE 의 최대 패킷 rate (64-byte): 1.488 Mpps.
- 10 GbE: 14.88 Mpps.
- 40 GbE: 59.52 Mpps.
- 100 GbE: 148.8 Mpps.
- 400 GbE: 595.2 Mpps.

### CPU 1 코어로 처리 가능한가?
- 일반 Linux TCP/IP stack: 패킷 당 ~1-3 μs.
- 1 core → 333K-1M pps 처리 가능.
- 100 GbE line rate (148 Mpps) = 100-400 코어 필요.

→ NIC 가 직접 처리하면 CPU 가 응용에만 집중.

## 2. Offload 기능 표

| 기능 | 의미 | 효과 |
| --- | --- | --- |
| **Checksum offload** | IP/TCP/UDP 체크섬 계산 / 검증 | CPU 0% 비용으로 검증 |
| **TSO (TCP Segmentation Offload)** | 큰 segment 를 NIC 가 잘라 보냄 | tx 코어 부담 ↓ |
| **GSO (Generic Segmentation Offload)** | OS 측 일반 segmentation | TSO 없는 NIC 도 효과 |
| **USO (UDP Segmentation Offload)** | UDP 의 segmentation | QUIC / 비디오 streaming |
| **LRO (Large Receive Offload)** | 작은 packets 를 큰 segment 로 stack 전달 | rx 코어 부담 ↓ |
| **GRO (Generic Receive Offload)** | OS 측 묶음 (LRO 의 software 버전) | 가상화 친화 |
| **VLAN tagging / stripping** | NIC 가 VLAN 헤더 처리 | |
| **TLS offload (kTLS)** | TLS 1.2/1.3 record 직접 | Netflix CDN, Cloudflare 사용 |
| **IPsec offload** | crypto + 인증 NIC 처리 | |
| **VXLAN / Geneve / NVGRE encap offload** | overlay 헤더 처리 | 클라우드 가상 네트워크 |
| **PTP / Hardware Timestamping** | 패킷 도착 / 출발 시각 ns 정밀 | 금융 / 5G |
| **RoCE (RDMA over Converged Ethernet)** | RDMA 직접 | AI 학습 / HPC |

## 3. Checksum Offload

### IP / TCP / UDP 헤더의 checksum
- IP 헤더 = simple 1's complement sum.
- TCP / UDP = pseudo-header + payload checksum.
- TCP segment 마다 계산 → 대용량 transfer 시 CPU 부담.

### NIC 가 처리
- TX: OS 가 0 placeholder 둔 후 NIC 가 채움.
- RX: NIC 가 검증 → OS 에 valid bit 전달.

```bash
sudo ethtool -k eth0 | grep -i checksum
# rx-checksumming: on
# tx-checksumming: on
# tx-checksum-ipv4: on
# tx-checksum-ip-generic: on
# tx-checksum-ipv6: on
# tx-checksum-fcoe-crc: on
# tx-checksum-sctp: on
```

### 비활성화 시
```bash
sudo ethtool -K eth0 rx-checksumming off
```

## 4. TSO (TCP Segmentation Offload)

### 동작
1. OS 가 큰 TCP segment 작성 (예: 64 KB).
2. NIC 에 큰 segment + meta (MSS, header template) 전달.
3. NIC 가 64 KB 를 MSS (1448 byte) 단위로 분할.
4. 각 segment 에 TCP/IP 헤더 자동 작성 + checksum.

### 효과
- 같은 throughput 에서 CPU 사용량 1/10 ~ 1/30.
- 대용량 download / upload (S3, video streaming) 의 표준.

### TSO 의 한계 / 부작용
- VM 호스트 / router 역할 시 packet 가 응용에 도달하기 전 단편화 → 일부 시나리오 문제.
- bufferbloat — 큰 segment 가 큐에 쌓여 latency tail ↑.

## 5. GSO (Generic Segmentation Offload)

### TSO 없는 NIC 에서도 효과
- OS 가 stack 마지막 (NIC driver 직전) 에서 segmentation.
- physical L4 segmentation 은 NIC 가 안 함.
- 효과: stack 안 packet 수 줄임 → buffer 관리 / lock 줄임.

## 6. LRO (Large Receive Offload)

### 동작
- NIC 가 들어오는 작은 packets 를 같은 flow 끼리 묶어서 stack 에 전달.
- 1 큰 packet 처리 = N 작은 packet 처리보다 빠름.

### 부작용 — Forwarding 환경
- router / bridge 역할에서 LRO 가 segment 합쳐버리면 forwarding 시 다시 분할 필요.
- packet structure 가 변형되어 일부 protocol (LRO 가 GRE/IPsec 모름) 동작 안 함.

→ **forwarding host 는 LRO off, GRO 만**.

## 7. GRO (Generic Receive Offload)

### LRO 의 software 버전
- OS stack 의 NAPI 안에서 동작.
- 같은 flow 의 작은 packets 가 NIC 에서 stack 으로 올라온 직후 묶음.
- forwarding 시에도 안전 (응용 layer 도달 전 분할 자동).

```bash
sudo ethtool -K eth0 lro on gro on
```

## 8. TLS Offload (kTLS)

### kTLS 가 처리하는 것
- Linux kernel TLS — application 이 setsockopt() 로 TLS key 전달 → kernel 또는 NIC 가 직접 암호화.
- 큰 file (CDN, Netflix video) 의 TLS record encryption / decryption 부담 ↓.

### NIC kTLS offload
- ConnectX-6 DX+ / Intel E810 가 지원.
- TLS 1.2 / 1.3 record 의 AES-GCM 직접.
- sendfile() + kTLS = zero-copy + offloaded encryption.

### 보안 우려
- 일부 구현은 키가 NIC 메모리에 머무름.
- 보안 요구 환경에서 검토 필요.

## 9. IPsec Offload

- ESP / AH header 의 crypto 와 인증.
- VPN gateway 의 핵심.
- AES-GCM, ChaCha20-Poly1305 등 가속.

## 10. Overlay 가상 네트워크 Offload

### VXLAN / Geneve / NVGRE
- 클라우드 가상 네트워크의 캡슐화.
- 안쪽 packet 의 checksum / TSO / RSS 가 외부 헤더에 묻혀버림.
- NIC 가 inner header 인식 → 정상 작동.

```bash
sudo ethtool -k eth0 | grep -E 'tx-udp_tnl|rx-udp_tnl'
# tx-udp_tnl-segmentation: on
# tx-udp_tnl-csum-segmentation: on
```

## 11. RSS / RPS / RFS / aRFS — 큐 분산

### RSS (Receive Side Scaling) — HW
- 들어오는 packet 의 5-tuple hash.
- NIC 가 hash → indirection table → 큐 인덱스.
- 같은 flow = 같은 큐 = 같은 CPU = cache 친화.
- Toeplitz hash 가 표준 (RFC 6052).

```bash
sudo ethtool -l eth0                    # 최대 / 현재 큐 수
sudo ethtool -L eth0 combined 16        # 큐 수 16
sudo ethtool -x eth0                    # RSS indirection table 보기
sudo ethtool -X eth0 equal 16           # 균등 분산
sudo ethtool -X eth0 hkey ...           # hash key 변경
```

### Hash polarization 문제
- 같은 hash 함수 + 같은 key 가 L3 (host 의 NIC) 와 L2 (switch 의 ECMP) 양쪽 사용 시 → 일부 path 만 hit.
- 해법: hash key 를 noth host / switch 가 다르게.

### RPS (Receive Packet Steering) — SW
- RSS 미지원 NIC 또는 추가 redirect.
- 첫 IRQ 후 OS 가 hash 로 다른 CPU 로 packet redirect.

```bash
echo 'ff' | sudo tee /sys/class/net/eth0/queues/rx-0/rps_cpus
```

### RFS (Receive Flow Steering)
- flow 가 어느 CPU 에서 처리되는지 추적.
- 다음 packet 을 그 CPU 의 큐로 → cache 더 잘 활용.

```bash
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

### aRFS (Accelerated RFS) — HW + SW
- NIC 의 ntuple filter (n-tuple hash → 큐 매핑) 동적 갱신.
- application 이 read/recvmsg 한 CPU 알면 NIC 에 알려서 다음 packet 그 큐로.

## 12. NAPI — Linux 네트워크의 핵심

자세히: [[../bus-io/dma-iommu]] §10.

요약:
- 첫 IRQ 후 polling 으로 일정 budget 까지 (보통 64 packet) 묶음.
- 패킷 폭주 시 polling 모드, 한가하면 interrupt.

## 13. Interrupt Coalescing

```bash
sudo ethtool -c eth0
# Default coalescing parameters:
# Adaptive RX: on  TX: on
# rx-usecs: 8
# rx-frames: 32
# tx-usecs: 8
# tx-frames: 32

# 더 모아서 (throughput 우선, latency 비싸짐)
sudo ethtool -C eth0 rx-usecs 50 rx-frames 32

# 가급적 즉시 처리 (latency 우선)
sudo ethtool -C eth0 rx-usecs 1 rx-frames 1
sudo ethtool -C eth0 adaptive-rx off
```

### Adaptive coalescing
- 트래픽 양에 따라 자동 조절.
- 낮은 트래픽 = 작은 coalescing.
- 높은 트래픽 = 큰 coalescing.

## 14. DPDK — 사용자 공간 high-speed NIC

### 개념
- 커널 우회 (kernel bypass).
- 사용자 공간 driver 가 NIC 직접 다룸.
- huge page + DMA pin + busy-loop polling.
- ns 단위 latency.

### 사용처
- 통신사 / CDN / 보안 어플라이언스.
- 라우터 / firewall.
- 옛 trading platform.

### 단점
- 한 코어 100% busy-loop.
- 호환성 / 운영 복잡 (NIC driver 직접 관리).
- 같은 NIC 에 다른 응용 못 씀.

### DPDK 흐름
1. NIC 를 kernel 에서 unbind → DPDK driver 로 bind.
2. DPDK PMD (Poll Mode Driver) 가 직접 ring buffer 다룸.
3. 응용이 packet 처리.
4. 끝나면 NIC 를 kernel 로 rebind.

```bash
# DPDK 의 hugepage
echo 2048 > /proc/sys/vm/nr_hugepages       # 2 MB 페이지 2048 개 = 4 GB

# NIC bind
sudo dpdk-devbind --bind=vfio-pci 0000:01:00.0

# 응용 실행
sudo ./testpmd -l 0-3 -n 4 -- -i

# 끝나면 rebind
sudo dpdk-devbind --bind=mlx5_core 0000:01:00.0
```

## 15. XDP (eXpress Data Path) + eBPF

### 개념
- 커널 안에서 NIC driver 받기 직전에 eBPF 프로그램 실행.
- packet drop / forward / redirect / pass 결정.
- DPDK 의 일부 사용 사례를 커널 안에서 처리.

### XDP 모드
- **XDP_DROP** — packet 즉시 drop.
- **XDP_PASS** — 정상 stack 통과.
- **XDP_TX** — 같은 NIC 로 다시 송신 (load balancer).
- **XDP_REDIRECT** — 다른 NIC / AF_XDP socket 으로.

### 사용처
- **Cilium** — Kubernetes 네트워크 + 보안.
- **Cloudflare Magic Firewall** — DDoS 방어.
- **Facebook Katran** — L4 load balancer.

### 코드 예 (간단한 DDoS filter)

```c
SEC("xdp")
int xdp_drop_tcp_syn(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    if (eth + 1 > data_end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if (ip + 1 > data_end) return XDP_PASS;
    if (ip->protocol != IPPROTO_TCP) return XDP_PASS;

    struct tcphdr *tcp = (void *)((u32 *)ip + ip->ihl);
    if (tcp + 1 > data_end) return XDP_PASS;

    if (tcp->syn && !tcp->ack) {
        // 추가 검증 (rate limit 등)
        return XDP_DROP;
    }
    return XDP_PASS;
}
```

```bash
# Load
sudo ip link set dev eth0 xdpgeneric obj xdp_filter.o sec xdp

# Unload
sudo ip link set dev eth0 xdpgeneric off
```

## 16. AF_XDP

- XDP 가 packet 을 사용자 공간으로 zero-copy 전달.
- DPDK 보다 커널 통합 좋고, XDP 보다 사용자 공간 자유.
- 2018 부터 Linux 메인라인.

```c
// 간단한 흐름
int xsk_fd = socket(AF_XDP, SOCK_RAW, 0);
struct sockaddr_xdp sxdp = {
    .sxdp_family = AF_XDP,
    .sxdp_ifindex = if_nametoindex("eth0"),
    .sxdp_queue_id = 0,
    .sxdp_flags = XDP_ZEROCOPY,
};
bind(xsk_fd, (struct sockaddr *)&sxdp, sizeof(sxdp));
// rings setup, packet receive ...
```

## 17. Hardware Timestamping + PTP

### Hardware Timestamping
- NIC 가 packet 도착 / 출발 시각을 **ns 정밀**으로 stamp.
- ethtool option:

```bash
sudo ethtool -T eth0
# Time stamping parameters for eth0:
# Capabilities:
#         hardware-transmit (SOF_TIMESTAMPING_TX_HARDWARE)
#         software-transmit (SOF_TIMESTAMPING_TX_SOFTWARE)
#         hardware-receive  (SOF_TIMESTAMPING_RX_HARDWARE)
#         software-receive  (SOF_TIMESTAMPING_RX_SOFTWARE)
#         hardware-raw-clock (SOF_TIMESTAMPING_RAW_HARDWARE)
```

### PTP (Precision Time Protocol, IEEE 1588)
- 네트워크 시계 동기.
- master clock → slave clock.
- offset / delay 측정 후 자체 시계 조정.
- 정밀도: 1 μs 이내 (typical), 100 ns 이내 (BC, boundary clock).

### NTP vs PTP

| | NTP | PTP |
| --- | --- | --- |
| 정밀도 | ms | μs ~ 100 ns |
| HW 지원 | 불필요 | HW timestamping NIC 필수 |
| 인프라 | 인터넷 / 회사 NTP server | switch 가 PTP-aware (BC/TC) |
| 사용 | 일반 서버 | 금융 / 5G / 자율주행 / 방송 |

### chrony / ptp4l

```bash
# PTP daemon
sudo ptp4l -i eth0 -m

# OS clock 을 PTP 와 동기
sudo phc2sys -s eth0 -w
```

## 18. NIC 의 큐 / IRQ 토폴로지

```
         NIC
          │
   ┌──────┼───────────────────┐
   │      │                   │
  RX큐0  RX큐1  ... RX큐n   ← MSI-X vector 별
  TX큐0  TX큐1  ... TX큐n
   │      │                   │
  IRQ0  IRQ1                  │
   │      │                   │
  CPU0  CPU1                  │
   │      │
   └──┬───┴───┴───            │
      │
   Application
```

### 권장 설정
1. NIC 큐 수 = nproc 또는 그 절반.
2. 각 큐 → 하나의 CPU 코어 (IRQ affinity).
3. RSS 가 들어오는 packet 을 hash 분산.
4. NAPI 가 polling 으로 batch 처리.

### NUMA-aware 설정
- NIC 가 NUMA node 0 에 attached → IRQ 도 node 0 코어로.
- 응용도 node 0 에 pin.
- cross-NUMA 메모리 접근 회피.

```bash
# NIC 의 NUMA node 확인
cat /sys/class/net/eth0/device/numa_node

# 그 node 의 CPU
lstopo --of console
```

## 19. NIC 진단 — cheatsheet

```bash
# 기본
ethtool -i eth0                    # driver / firmware
ethtool eth0                       # link / speed / duplex
ethtool -S eth0                    # 통계 (drop / csum error / queue)
ethtool -k eth0                    # offload 기능 on/off
ethtool -K eth0 gro on tso on      # 켜기
ethtool -l eth0                    # 큐 수
ethtool -L eth0 combined 16        # 큐 수 변경
ethtool -c eth0                    # coalescing
ethtool -T eth0                    # timestamping

# 큐 별 통계
ethtool -S eth0 | grep -E 'rx_queue_[0-9]+_packets|tx_queue_[0-9]+_packets'

# packet drop / 큐 overflow
ethtool -S eth0 | grep -i drop
ethtool -S eth0 | grep -i discard

# 인터럽트
cat /proc/interrupts | grep eth0

# 처리 stats
ip -s link
sar -n DEV 1
mpstat -P ALL 1                    # CPU 별 softirq 시간 (%soft)

# Packet capture
sudo tcpdump -i eth0 -w out.pcap   # 일반
sudo ip -s link show eth0          # drop / error 카운터

# Flow steering (aRFS)
cat /proc/sys/net/core/rps_sock_flow_entries

# QoS / traffic control
tc -s qdisc show dev eth0
```

## 20. 모니터링 도구

### Prometheus / node_exporter
```yaml
- name: node_network
  rules:
  - alert: HighNetworkDropRate
    expr: rate(node_network_receive_drop_total[5m]) > 100
  - alert: ChecksumErrors
    expr: rate(node_network_receive_errs_total[5m]) > 10
```

### bpftrace
```bash
sudo bpftrace -e 'tracepoint:net:net_dev_xmit { @[args->name] = count(); }'
sudo bpftrace -e 'tracepoint:net:netif_receive_skb { @[args->name] = count(); }'
```

## 21. 함정

1. **RSS 미사용** — 100 GbE 가 단일 코어 100%, 전체 NIC drop.
2. **GRO/LRO 의 forwarding 문제** — router / bridge 역할에서 LRO 가 packet 변형. forwarding 서버는 LRO off.
3. **TLS offload 의 키 노출** — 일부 구현은 NIC 메모리에 키 머무름. 보안 요구 검토.
4. **DPDK 와 일반 NIC driver 혼용** — DPDK 가 NIC 점유 시 ping 도 못 나감.
5. **interrupt coalescing 너무 공격적** — latency 민감 응용에 100 μs+ 추가.
6. **MTU 9000 + offload** — 일부 NIC 의 TSO/GRO 가 jumbo 에서 버그.
7. **VXLAN / Geneve offload 미활성** — overlay 가상 네트워크 throughput 1/3.
8. **NIC NUMA node 무시** — cross-NUMA 메모리 접근으로 throughput ↓.
9. **PTP 없이 financial trading** — μs 단위 latency 측정 불가.
10. **AF_XDP 와 DPDK 혼동** — 같이 못 씀. 하나 골라야.
11. **XDP 의 packet size 제한** — XDP_TX 시 외부 헤더 추가 못 함 (메모리 추가 reserve 필요).
12. **eBPF verifier 의 한계** — 복잡한 logic 은 verifier 가 reject. 코드 단순화.

## 22. 관련

- [[network-hardware]]
- [[switch-fabric]]
- [[rdma-infiniband]] — RDMA / RoCE
- [[../bus-io/dma-iommu]] — MSI-X / NAPI / RSS / SR-IOV
- [[../../network/network]] — 프로토콜 레이어
- [[../../operating-system/operating-system]] — softirq, packet path, eBPF
