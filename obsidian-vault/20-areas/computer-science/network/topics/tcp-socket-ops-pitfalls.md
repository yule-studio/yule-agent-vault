---
title: "TCP socket 운영 pitfall — TIME_WAIT / SO_REUSEPORT / ephemeral port / SYN backlog"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T15:30:00+09:00
tags: [network, topics, tcp, socket, sysctl, devops]
home_hub: network
related:
  - "[[topics]]"
  - "[[../network]]"
  - "[[../tcp/tcp-state-machine]]"
  - "[[../tcp/three-way-handshake]]"
  - "[[../tools/ss-netstat-tcp-tools]]"
  - "[[../tools/iptables-netfilter]]"
  - "[[../../../devops/linux/linux-mental-models-for-devops]]"
---

# TCP socket 운영 pitfall — TIME_WAIT / SO_REUSEPORT / ephemeral port / SYN backlog

**[[topics|↑ topics]]**

---

## 1. 목적

본 문서는 high-concurrency 환경 (load balancer / API gateway / reverse proxy / connection-heavy 서버) 에서 자주 부딪히는 TCP socket 운영 이슈를 정의한다.

본 문서가 정의하는 것:
- BSD socket 의 상태 전이 (TIME_WAIT / CLOSE_WAIT / FIN_WAIT)
- TIME_WAIT 의 의미와 의도된 동작
- SO_REUSEADDR vs SO_REUSEPORT
- ephemeral port range 와 고갈
- SYN backlog 와 accept queue
- conntrack table 한계
- 각 항목의 sysctl tuning 과 application 책임의 경계

본 문서가 정의하지 않는 것:
- TCP 프로토콜 자체 — [[../tcp/tcp]]
- 3-way handshake 동작 — [[../tcp/three-way-handshake]]
- 혼잡 제어 — [[../tcp/congestion-control]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 OS | Linux (kernel 4.x+) |
| 대상 워크로드 | HTTP 서버 / LB / reverse proxy / DB connection pool / outgoing API client |
| 제외 | application 프레임워크별 connection pool 구현 |
| 제외 | UDP socket — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **socket** | (protocol, local_ip, local_port, remote_ip, remote_port) 의 5-tuple. |
| **listening socket** | 서버가 bind + listen 한 socket. accept 로 새 연결 socket 생성. |
| **established socket** | 양방향 연결이 성립된 socket. |
| **TIME_WAIT** | active close 한 쪽이 마지막 ACK 후 일정 시간 (2×MSL ≈ 60s) 유지하는 상태. |
| **ephemeral port** | client 가 outgoing 연결 시 OS 가 자동 할당하는 source port. |
| **SO_REUSEADDR** | 같은 (IP, port) 를 짧은 시간 안에 다시 bind 허용. |
| **SO_REUSEPORT** | 같은 (IP, port) 를 여러 listening socket 이 동시 bind 허용 (kernel 이 load balance). |
| **SYN backlog** | SYN_RCVD 상태의 half-open 연결 큐. |
| **accept queue** | ESTABLISHED 상태 중 아직 accept() 호출 안 된 연결 큐. |

---

## 4. socket 의 상태 전이 (요약)

```
                       LISTEN
                          │
                          │ SYN 받음
                          ▼
                      SYN_RCVD
                          │
                          │ ACK 받음
                          ▼
                      ESTABLISHED
                       /        \
            active close        passive close
                 │                    │
                 ▼                    ▼
            FIN_WAIT_1            CLOSE_WAIT
                 │                    │
                 ▼                    ▼
            FIN_WAIT_2            LAST_ACK
                 │                    │
                 ▼                    ▼
            TIME_WAIT             CLOSED
                 │
                 ▼ (2×MSL 후)
              CLOSED
```

상세: [[../tcp/tcp-state-machine]], [[../tcp/four-way-termination]].

---

## 5. TIME_WAIT — 의도된 60초

### 5.1 정의

active close 한 쪽 (먼저 FIN 보낸 쪽) 이 마지막 ACK 송신 후 **2×MSL** (Linux default 60s) 동안 socket 을 점유.

### 5.2 왜 필요한가

| 이유 | 효과 |
| --- | --- |
| 마지막 ACK 가 lost 시 재전송 대응 | 상대의 FIN 재전송에 ACK 응답 가능 |
| 같은 (5-tuple) 의 옛 packet 이 새 연결과 섞이지 않음 | 망 안의 잔여 packet 이 새 연결에 inject 되는 사고 방지 |

### 5.3 운영 영향

| 환경 | 영향 |
| --- | --- |
| client (outgoing) | 같은 dest 로 많은 short-lived 연결 → ephemeral port 고갈 (§7) |
| server (incoming) | TIME_WAIT 가 socket 자체는 차지하지만 listening 과 충돌 X — 보통 무해 |
| LB ↔ backend | LB 가 active close 시 LB 측 TIME_WAIT 누적 |

### 5.4 sysctl

```bash
# 현황
ss -ant state time-wait | wc -l
sysctl net.ipv4.tcp_fin_timeout       # default 60

# TIME_WAIT 재사용 (outgoing 만 영향)
sysctl -w net.ipv4.tcp_tw_reuse=1

# (deprecated) tcp_tw_recycle
# - kernel 4.12 부터 제거됨. NAT 뒤에서 패킷 drop 유발하던 옵션.
```

→ `tcp_tw_reuse=1` 은 outgoing 연결의 TIME_WAIT 재사용을 허용. 일반적으로 안전.

→ `tcp_tw_recycle` 은 사용 금지 (kernel 4.12 제거).

---

## 6. CLOSE_WAIT — application 책임

### 6.1 정의

상대가 FIN 을 보냈는데 (passive close 의 시작) 자기 application 이 `close()` 호출 안 한 상태.

### 6.2 의미

| 누적 | 해석 |
| --- | --- |
| TIME_WAIT 누적 | OS 의 의도된 동작 (보통 무해) |
| CLOSE_WAIT 누적 | application bug — close 누락 / 연결 누갈 |

### 6.3 추적

```bash
ss -ant state close-wait
ss -ant state close-wait | awk '{print $5}' | sort | uniq -c
lsof -i -P | grep CLOSE_WAIT

# process 별
for pid in $(pgrep -f myapp); do
  echo "pid=$pid close_wait=$(ls /proc/$pid/fd | wc -l)"
done
```

→ CLOSE_WAIT 가 증가하면 application 의 connection 처리 코드 (try/finally, defer close, autoCloseable) 점검.

---

## 7. ephemeral port 고갈

### 7.1 정의

client 가 outgoing connect 시 OS 가 source port 를 자동 할당. Linux 기본 range 는 `32768-60999` (≈ 28000 개).

### 7.2 고갈 시나리오

| 시나리오 | 결과 |
| --- | --- |
| 한 dest (IP+port) 로 28000+ 연결 동시 | port 고갈 → connect 실패 (`EADDRNOTAVAIL`) |
| 짧은 lifetime 연결 폭주 + TIME_WAIT | port 가 60s 동안 묶임 |
| LB 가 backend 로 connection pool 없이 매번 새 연결 | 가장 흔한 사례 |
| Docker / k8s 의 외부 호출 폭주 | host 의 ephemeral port 공유 |

### 7.3 sysctl

```bash
# 현황
sysctl net.ipv4.ip_local_port_range
# 32768  60999

# 확장
sysctl -w net.ipv4.ip_local_port_range="10240 65535"

# TIME_WAIT 재사용 (§5.4)
sysctl -w net.ipv4.tcp_tw_reuse=1

# 같은 dest 로 가는 연결 수 추적
ss -ant | awk '$5 ~ /:443$/' | wc -l
```

### 7.4 근본 해결

- **connection pool 사용** — 매 호출마다 새 연결 대신 재사용
- **HTTP/2 / gRPC** — multiplexing 으로 적은 연결 다수 stream
- **여러 source IP** — host 에 secondary IP 추가
- **여러 dest port** — 가능하면 LB 가 여러 port 노출

---

## 8. SO_REUSEADDR vs SO_REUSEPORT

### 8.1 SO_REUSEADDR

같은 (local_ip, local_port) 를 짧은 시간 안에 다시 bind 허용. 주로 서버 재시작 시 TIME_WAIT 상태의 socket 때문에 "Address already in use" 막는 용도.

```c
int opt = 1;
setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

→ 대부분의 production 서버 코드가 사용. 위험 없음.

### 8.2 SO_REUSEPORT

같은 (local_ip, local_port) 를 여러 listening socket 이 **동시** bind. kernel 이 incoming 연결을 socket 간 hash 로 load balance.

```c
int opt = 1;
setsockopt(s, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
```

### 8.3 SO_REUSEPORT 의 사용

| 사용 | 효과 |
| --- | --- |
| multi-worker 서버 (nginx worker_processes) | 각 worker 가 같은 port 에 listen → kernel 이 분배 |
| graceful restart | 신/구 process 가 잠시 동시 listen |
| k8s Service externalTrafficPolicy=Local | 노드별 backend pod 의 균등 분배 |

### 8.4 차이

| 항목 | SO_REUSEADDR | SO_REUSEPORT |
| --- | --- | --- |
| 목적 | TIME_WAIT 의 옛 socket 회피 | 여러 socket 의 동시 listen + LB |
| 동시 listen | 1 (옛 socket 이 TIME_WAIT) | 다수 |
| kernel 분배 | 없음 | hash (default) 또는 BPF (kernel 4.5+) |

---

## 9. SYN backlog 와 accept queue

### 9.1 두 큐

```
SYN 들어옴
   ↓
SYN_RCVD (half-open)        ← syn_backlog (net.ipv4.tcp_max_syn_backlog)
   ↓ (ACK 받음)
ESTABLISHED                  ← accept queue (listen(backlog))
   ↓ (application accept())
application 처리
```

| 큐 | 한도 | 초과 시 |
| --- | --- | --- |
| syn_backlog | `net.ipv4.tcp_max_syn_backlog` | SYN drop 또는 SYN cookies |
| accept queue | `listen(fd, backlog)` 의 backlog 와 `net.core.somaxconn` 중 작은 값 | 새 연결 drop |

### 9.2 sysctl

```bash
# syn backlog
sysctl net.ipv4.tcp_max_syn_backlog       # default 128 ~ 256 (작음)
sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# accept queue 상한
sysctl net.core.somaxconn                  # default 4096 (구버전 128)
sysctl -w net.core.somaxconn=65535

# SYN cookies (SYN flood 보호)
sysctl net.ipv4.tcp_syncookies              # default 1
```

→ application 의 `listen(fd, backlog)` 호출에서 backlog 를 `somaxconn` 이상으로 줘도 truncate 됨.

### 9.3 현황 확인

```bash
ss -lnt
# Recv-Q Send-Q  Local Address:Port   Peer Address:Port
#    0      4096      *:80                  *:*
#  ^^^^   ^^^^
#  현재    accept queue 한도
#  미수락
#  연결 수

# 통계
netstat -s | grep -iE "listen|drop|overflow"
# - "TCP: x SYNs to LISTEN sockets ignored"   → SYN drop
# - "x times the listen queue of a socket overflowed"  → accept queue 초과
```

→ accept queue overflow 가 증가하면 application 이 `accept()` 를 충분히 빠르게 못 부르고 있음. worker 수 / async 처리 점검.

---

## 10. conntrack table 의 영향

netfilter conntrack 이 TCP 연결마다 entry 를 생성. high-concurrency 노드 / NAT gateway 에서 한도 초과 시 packet drop.

```bash
# 현황 / 한도
sysctl net.netfilter.nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count

# TCP timeout
sysctl net.netfilter.nf_conntrack_tcp_timeout_established    # default 432000 (5 day)
sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait      # default 120

# drop 통계
conntrack -S
```

상세: [[../tools/iptables-netfilter]] §7.

---

## 11. 항목별 sysctl 요약

| 항목 | 권장 |
| --- | --- |
| `net.ipv4.ip_local_port_range` | `"10240 65535"` (outgoing 폭주) |
| `net.ipv4.tcp_tw_reuse` | `1` (outgoing) |
| `net.ipv4.tcp_fin_timeout` | `30` (선택) |
| `net.core.somaxconn` | `65535` (모든 서버) |
| `net.ipv4.tcp_max_syn_backlog` | `8192` 이상 |
| `net.ipv4.tcp_syncookies` | `1` (default 유지) |
| `net.netfilter.nf_conntrack_max` | 호스트 메모리 / 1KB 당 entry (보통 수십만) |
| `net.netfilter.nf_conntrack_tcp_timeout_established` | `1800` (기본 5일은 너무 김) |
| `net.ipv4.tcp_keepalive_time` | `300` (default 7200 은 너무 김) |

→ sysctl 변경은 `/etc/sysctl.d/99-tcp-tuning.conf` 에 영구 저장.

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 모델 설명 |
| --- | --- | --- |
| client 에서 `connect: cannot assign requested address` | ephemeral port 고갈 | §7 |
| TIME_WAIT 가 수만 개 보임 | active close 폭주 — 보통 정상, 단 ephemeral port 고갈 동반 시 문제 | §5 |
| CLOSE_WAIT 계속 증가 | application 의 close 누락 | §6 |
| 부하 시 일부 client connect refused | accept queue overflow | §9 |
| 서버 재시작 시 "Address already in use" | SO_REUSEADDR 미사용 + TIME_WAIT | §8.1 |
| nginx worker 가 골고루 못 분산 | SO_REUSEPORT 미사용 | §8.2 |
| 노드의 packet drop, dmesg 에 `nf_conntrack: table full` | conntrack 한도 | §10 |
| TCP 연결이 절대 끊기지 않음 (좀비) | keepalive 미설정 + conntrack timeout 5일 | §11 |
| LB 의 backend 응답 점점 느려짐 | LB 가 backend 마다 매번 새 연결 (HTTP keep-alive off) | §7.4 |
| `tcp_tw_recycle` 켰더니 NAT 뒤 client 가 packet drop | deprecated 옵션, 사용 금지 | §5.4 |

---

## 13. 참고

- [[topics|↑ topics]]
- [[../network|↑ network hub]]
- [[../tcp/tcp-state-machine]] — 상태 전이 상세
- [[../tcp/three-way-handshake]] — SYN / SYN-ACK / ACK
- [[../tcp/four-way-termination]] — FIN / ACK / FIN / ACK
- [[../tcp/tcp-keepalive]] — keepalive 메커니즘
- [[../tcp/tcp-performance-tuning]] — 추가 sysctl
- [[../tools/ss-netstat-tcp-tools]] — ss / netstat 사용
- [[../tools/iptables-netfilter]] — conntrack
- [[../../../devops/linux/linux-mental-models-for-devops]] — process / fd
- man 7 tcp, man 7 socket
- kernel docs — `Documentation/networking/ip-sysctl.rst`
