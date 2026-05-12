---
title: "TCP 성능 튜닝 (Performance Tuning)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:05:00+09:00
tags:
  - network
  - tcp
  - performance
  - tuning
  - sysctl
---

# TCP 성능 튜닝 (Performance Tuning)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | sysctl / buffer / BDP / NIC / fq / 디버깅 |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 튜닝의 4 축

1. **버퍼 크기** (BDP 만큼)
2. **혼잡 제어 알고리즘** (CUBIC / BBR)
3. **NIC offload** (TSO / GRO / RSS)
4. **시스템 자원** (file descriptor / port range / connection backlog)

---

## 2. BDP — Bandwidth-Delay Product

```
BDP = Bandwidth × RTT
```

### 2.1 BDP 별 권장 buffer

| 시나리오 | Bandwidth | RTT | BDP | 권장 buffer |
| --- | --- | --- | --- | --- |
| LAN | 1 Gbps | 1 ms | 125 KB | 256 KB |
| Cross-city | 1 Gbps | 30 ms | 3.75 MB | 8 MB |
| Cross-country | 1 Gbps | 100 ms | 12.5 MB | 16 MB |
| KR ↔ US | 1 Gbps | 200 ms | 25 MB | 32 MB |
| 위성 (GEO) | 10 Mbps | 600 ms | 750 KB | 2 MB |
| 10G + 100ms | 10 Gbps | 100ms | 125 MB | 256 MB |

---

## 3. Linux sysctl 튜닝

### 3.1 Buffer 크기

```bash
# TCP receive buffer (min default max)
net.ipv4.tcp_rmem = 4096 87380 33554432    # max 32MB
net.ipv4.tcp_wmem = 4096 65536 33554432

# Socket buffer 최대 (rmem_max / wmem_max)
net.core.rmem_max = 67108864     # 64 MB
net.core.wmem_max = 67108864
net.core.rmem_default = 262144
net.core.wmem_default = 262144

# Buffer 자동 튜닝
net.ipv4.tcp_moderate_rcvbuf = 1
```

### 3.2 혼잡 제어 / 옵션

```bash
# 알고리즘
net.ipv4.tcp_congestion_control = bbr   # 또는 cubic

# Window Scale / Timestamp / SACK
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_sack = 1
net.ipv4.tcp_fack = 1

# Initial Window (RFC 6928)
net.ipv4.tcp_init_cwnd = 10   # 기본 10

# ECN
net.ipv4.tcp_ecn = 2   # 2 = 수동적 (안전)
```

### 3.3 Backlog / Queue

```bash
# Listen queue (accept queue)
net.core.somaxconn = 65535

# SYN queue
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_syncookies = 1

# Backlog (input queue)
net.core.netdev_max_backlog = 30000
```

### 3.4 TIME_WAIT / Connection 재사용

```bash
# TIME_WAIT 재사용
net.ipv4.tcp_tw_reuse = 1
# net.ipv4.tcp_tw_recycle = 0    # 제거됨 (4.12+) — 절대 X

# 빠른 종료
net.ipv4.tcp_fin_timeout = 30   # 기본 60 → 30
```

### 3.5 Keepalive

```bash
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
```

### 3.6 Port range

```bash
net.ipv4.ip_local_port_range = "10240 65535"
```

### 3.7 적용

```bash
# 임시
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 영구
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.d/99-tcp.conf
sudo sysctl -p /etc/sysctl.d/99-tcp.conf
```

---

## 4. qdisc / Pacing (BBR 의 필수)

```bash
# 현재 qdisc
tc qdisc show

# BBR 위한 fq (Fair Queue + pacing)
sudo tc qdisc replace dev eth0 root fq

# CAKE — 더 똑똑한 AQM
sudo tc qdisc replace dev eth0 root cake bandwidth 100mbit
```

| qdisc | 용도 |
| --- | --- |
| **pfifo_fast** | 옛 기본, FIFO |
| **fq_codel** | 모던 기본 (Linux 5.5+) |
| **fq** | BBR 의 pacing 친구 |
| **cake** | 가정 / 라우터 |

---

## 5. NIC Offload

### 5.1 보기 / 설정

```bash
# 현재 offload 기능
ethtool -k eth0

# 설정
sudo ethtool -K eth0 tso on
sudo ethtool -K eth0 gso on
sudo ethtool -K eth0 gro on
sudo ethtool -K eth0 rx-checksumming on
sudo ethtool -K eth0 tx-checksumming on
```

### 5.2 주요 offload

| 기능 | 의미 |
| --- | --- |
| **TSO** (TCP Segmentation Offload) | OS 가 큰 segment → NIC 가 자름 |
| **GSO** (Generic Segmentation Offload) | TSO 의 일반화 (UDP 도) |
| **LRO** (Large Receive Offload) | 수신 시 합침 |
| **GRO** (Generic Receive Offload) | LRO 의 일반화, 안전 |
| **RSS** (Receive Side Scaling) | 여러 CPU 코어 분산 |
| **RPS** (Receive Packet Steering) | SW 버전 RSS |
| **XPS** (Transmit Packet Steering) | 송신 분산 |
| **Checksum offload** | CRC HW 계산 |

### 5.3 IRQ 분산

```bash
# 현재 IRQ
cat /proc/interrupts | grep eth0

# 분산
sudo systemctl enable --now irqbalance

# 또는 수동
echo 0 > /proc/irq/24/smp_affinity_list
```

---

## 6. 응용 레벨 튜닝

### 6.1 TCP_NODELAY (Nagle 끄기)
```c
int yes = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes));
```
실시간 응용 (게임, RPC) 권장.

### 6.2 TCP_QUICKACK
```c
int yes = 1;
setsockopt(sock, IPPROTO_TCP, TCP_QUICKACK, &yes, sizeof(yes));
```
Delayed ACK 끄기 — Nagle 함정 회피.

### 6.3 SO_REUSEADDR / SO_REUSEPORT
- 빠른 재시작 / 부하 분산

### 6.4 큰 송수신 버퍼
```c
int size = 4 * 1024 * 1024;
setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
```

---

## 7. 파일 디스크립터 / 자원

```bash
# 현재 한계
ulimit -n
cat /proc/sys/fs/file-max

# 설정
ulimit -n 1048576

# 영구 (/etc/security/limits.conf)
* soft nofile 1048576
* hard nofile 1048576

# systemd
[Service]
LimitNOFILE=1048576
```

---

## 8. 진단 명령

### 8.1 ss

```bash
# 상세 TCP 정보
ss -tin

# 출력 예:
# cubic wscale:7,7 rto:204 rtt:7.234/5.123
#   mss:1460 pmtu:1500 rcvmss:1448 advmss:1460
#   cwnd:50 ssthresh:30 bytes_sent:1234567 segs_out:5000
#   send 800.4Mbps lastsnd:120 lastrcv:120 lastack:120
#   pacing_rate 1.6Gbps delivery_rate 800Mbps app_limited
#   busy:5000ms rwnd_limited:0ms sndbuf_limited:0ms
```

### 8.2 nstat

```bash
nstat -a    # 모든 카운터
nstat       # 변화만
```

주요 카운터:
- `TcpExtTCPRetransSegs` — 재전송 segment 수
- `TcpExtTCPDSACKRecv` — DSACK 받은 수 (네트워크 손실)
- `TcpExtTCPCongestionExperiencedECN` — ECN 신호

### 8.3 iftop / nload / sar
실시간 throughput.

```bash
sudo iftop -i eth0
nload eth0
sar -n DEV 1
```

### 8.4 wrk / iperf3

```bash
# HTTP 부하 측정
wrk -t12 -c400 -d30s https://example.com/

# 대역폭 측정
# 서버
iperf3 -s
# 클라이언트
iperf3 -c server -t 30 -P 8
```

---

## 9. 흔한 함정과 처방

### 함정 1 — 큰 BDP + 작은 buffer
**증상**: throughput 가 낮음
**진단**: `ss -tin` 의 `snd_wnd`, `cwnd` 가 작음, `rwnd_limited` > 0
**처방**: `tcp_rmem` / `tcp_wmem` / `rmem_max` 증가

### 함정 2 — Bufferbloat
**증상**: throughput 는 OK 인데 latency 폭증
**진단**: ping latency 가 부하 시 증가
**처방**: fq_codel / cake qdisc, BBR 혼잡 제어

### 함정 3 — 작은 backlog
**증상**: 새 연결 거부
**진단**: `ss -ltn` 의 Send-Q 가 가득
**처방**: `somaxconn`, listen() backlog 증가

### 함정 4 — TIME_WAIT 폭증
**증상**: 새 연결 못 만듦 (ephemeral port 소진)
**진단**: `ss -tan | grep TIME-WAIT | wc -l`
**처방**: Connection pool, `tcp_tw_reuse=1`, 포트 범위 늘림

### 함정 5 — NIC IRQ 한 코어 집중
**증상**: 1 CPU 100%, 나머지 0%
**진단**: `mpstat -P ALL 1`
**처방**: RSS, irqbalance, multi-queue NIC

### 함정 6 — 작은 IW
**증상**: 짧은 응답 느림
**진단**: cwnd 작음
**처방**: `initcwnd 10` (route 별) 또는 sysctl

---

## 10. 데이터센터 환경

### 10.1 DCTCP
- Data Center TCP (RFC 8257)
- ECN 기반 미세 cwnd 조정
- 인캐스트 (incast) 문제 해결

```bash
net.ipv4.tcp_congestion_control = dctcp
net.ipv4.tcp_ecn = 1
```

### 10.2 RDMA / RoCE
- TCP 보다 빠른 메모리 직접 전송
- L1-L2 가 lossless 필요 (PFC)

자세히 → [[../osi-7-layer/layer-2-data-link/flow-control#7 802.3x PAUSE Frame]]

---

## 11. 학습 자료

- "Linux TCP/IP Tuning" — Cilium / Cloudflare 백서
- "Making applications faster" — High Performance Browser Networking
- Linux Kernel Documentation `Documentation/networking/`
- BPF perf — `bpftrace` 로 TCP 행동 추적

---

## 12. 관련

- [[sliding-window]] — Buffer / Window
- [[congestion-control]] — BBR / CUBIC
- [[tcp-keepalive]] — Idle 처리
- [[tcp]] — TCP hub
