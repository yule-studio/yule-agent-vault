---
title: "iptables / nftables / netfilter — Linux packet filter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T14:00:00+09:00
tags: [network, tools, iptables, nftables, netfilter, firewall, linux]
home_hub: network
related:
  - "[[tools]]"
  - "[[../network]]"
  - "[[../ip/nat]]"
  - "[[container-networking]]"
  - "[[../../../devops/networking-ops/network-policy]]"
  - "[[../../../devops/linux/linux-mental-models-for-devops]]"
---

# iptables / nftables / netfilter — Linux packet filter

**[[tools|↑ tools]]**

---

## 1. 목적

본 문서는 Linux 커널의 packet filter 프레임워크인 **netfilter** 와 그 사용자 공간 도구 (**iptables**, **nftables**) 를 정의한다.

본 문서가 정의하는 것:
- netfilter 의 hook point 와 처리 흐름
- iptables 의 4 table × 5 chain 모델
- nftables 의 통합 syntax 와 iptables 차이
- conntrack 의 상태 추적 모델
- DevOps 도구 (Docker / kube-proxy / firewalld / fail2ban) 가 netfilter 를 사용하는 방식
- 디버깅 절차

본 문서가 정의하지 않는 것:
- 응용 계층 WAF — [[../../../devops/networking-ops/waf]]
- k8s NetworkPolicy 의 spec / 의미 — [[../../../devops/networking-ops/network-policy]]
- eBPF 기반 packet 처리 — [[../../../devops/networking-ops/cni-deep]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 커널 | Linux 3.13 이상 (nftables 도입). 5.0 이상 권장. |
| 대상 도구 | iptables (legacy + nf_tables backend), nftables (nft), ip6tables, ebtables, arptables |
| 제외 | BSD pf, Windows Defender Firewall |
| 제외 | eBPF / XDP — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **netfilter** | Linux 커널의 packet filter / NAT 프레임워크. hook point 위에 모듈을 등록한다. |
| **hook point** | packet 이 커널 network stack 의 특정 위치를 지날 때 호출되는 callback 등록 지점. |
| **iptables** | netfilter 위에 만들어진 사용자 공간 도구. table / chain / rule 으로 packet 처리 규칙을 표현한다. |
| **nftables (nft)** | iptables / ip6tables / ebtables / arptables 를 단일 syntax 로 통합한 후속 도구. backend 도 nf_tables 로 통합. |
| **table** | 같은 종류의 처리 (filter / nat / mangle / raw / security) 를 묶는 단위. |
| **chain** | packet 이 hook point 에서 평가되는 rule 의 순차 list. |
| **rule** | match 조건 + target (ACCEPT / DROP / REJECT / SNAT / DNAT / LOG / RETURN 등) 의 한 줄. |
| **conntrack** | netfilter 의 connection tracking 모듈. flow 별 state (NEW / ESTABLISHED / RELATED / INVALID) 추적. |

---

## 4. netfilter hook point

packet 이 커널 network stack 을 통과할 때 다음 5 지점에서 callback 이 호출된다.

```
                ┌────────────┐
                │ NIC (eth0) │
                └─────┬──────┘
                      ▼
                ┌────────────┐
                │ PREROUTING │   ← incoming packet 의 최초 hook
                └─────┬──────┘
                      ▼
              [routing decision]
            local?  ─┐         ─ forward?
                     ▼                       ▼
            ┌────────────┐         ┌──────────┐
            │   INPUT    │         │ FORWARD  │
            └─────┬──────┘         └────┬─────┘
                  ▼                       │
            local process                  │
                  │                       │
                  ▼                       │
            ┌────────────┐                 │
            │   OUTPUT   │                 │
            └─────┬──────┘                 │
                  ▼                       │
              [routing decision]            │
                     │                     │
                     └─────────┬───────────┘
                               ▼
                       ┌─────────────┐
                       │ POSTROUTING │   ← outgoing packet 의 최종 hook
                       └─────┬───────┘
                             ▼
                       ┌────────────┐
                       │ NIC (eth0) │
                       └────────────┘
```

| hook | 호출 시점 |
| --- | --- |
| PREROUTING | NIC 수신 직후 (라우팅 결정 전) |
| INPUT | local process 로 향하는 packet |
| FORWARD | 다른 인터페이스로 forwarding 되는 packet |
| OUTPUT | local process 가 생성한 packet |
| POSTROUTING | NIC 송신 직전 (라우팅 결정 후) |

---

## 5. iptables 의 4 table × 5 chain 모델

### 5.1 table 정의

| table | 목적 | 기본 chain |
| --- | --- | --- |
| **filter** | packet 허용 / 차단 | INPUT / FORWARD / OUTPUT |
| **nat** | 주소 / 포트 변환 | PREROUTING / OUTPUT / POSTROUTING |
| **mangle** | header 변경 (TTL / TOS / MARK) | 5 chain 모두 |
| **raw** | conntrack 우회 (NOTRACK) | PREROUTING / OUTPUT |
| **security** | SELinux mark | INPUT / FORWARD / OUTPUT |

### 5.2 처리 순서 (incoming packet, local destination)

```
PREROUTING (raw → conntrack → mangle → nat-DNAT)
   ↓
[routing decision]
   ↓ (local)
INPUT (mangle → filter → security)
   ↓
local process
```

### 5.3 처리 순서 (forwarded packet)

```
PREROUTING (raw → mangle → nat-DNAT)
   ↓
[routing decision]
   ↓ (forward)
FORWARD (mangle → filter → security)
   ↓
POSTROUTING (mangle → nat-SNAT)
   ↓
NIC out
```

### 5.4 명령 예

```bash
# 상태 조회
iptables -t filter -L INPUT -n -v --line-numbers
iptables -t nat -L -n -v

# 22 포트 incoming 허용 (filter)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 모든 incoming 차단 (default deny)
iptables -P INPUT DROP

# established / related 는 통과
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# DNAT — 80 으로 들어온 packet 을 내부 10.0.0.5:8080 으로
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080

# SNAT (MASQUERADE) — 외부 나가는 packet 의 source 를 eth0 IP 로
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 저장 / 복원
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

---

## 6. nftables 의 통합

nftables 는 iptables / ip6tables / ebtables / arptables 를 단일 syntax 로 통합한 후속 도구이다.

### 6.1 차이

| 항목 | iptables | nftables |
| --- | --- | --- |
| binary | iptables / ip6tables / ebtables / arptables | nft 1 개 |
| backend | x_tables (legacy) 또는 nf_tables | nf_tables |
| syntax | 위치 의존 옵션 (`-A INPUT -p tcp --dport 22 -j ACCEPT`) | 표현식 (`add rule ip filter input tcp dport 22 accept`) |
| atomic update | 부분 update 만 가능 (rule 단위) | table 전체 atomic replace 가능 |
| set / map | ipset 별도 도구 필요 | 내장 (named set / map) |
| 성능 | 룰 수 증가 시 O(N) | set 사용 시 O(log N) 또는 O(1) |

### 6.2 예

```bash
# table 생성 + chain + rule
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input iif lo accept

# named set 으로 IP 그룹 차단
nft add set inet filter blocklist { type ipv4_addr \; }
nft add element inet filter blocklist { 1.2.3.4, 5.6.7.8 }
nft add rule inet filter input ip saddr @blocklist drop
```

→ 신규 환경은 nftables 가 표준. 기존 도구 (docker / firewalld / k8s) 는 backend 를 nf_tables 로 사용하면서 iptables wrapper 호출도 유지.

---

## 7. conntrack — connection tracking

### 7.1 정의

netfilter 는 packet 별이 아니라 **flow 별** 로 state 를 추적한다. state 는 NEW / ESTABLISHED / RELATED / INVALID / UNTRACKED.

```bash
# conntrack table 조회
conntrack -L
# 예: tcp 6 431999 ESTABLISHED src=10.0.0.1 dst=8.8.8.8 sport=54321 dport=443 ...

# 통계
conntrack -S
cat /proc/net/nf_conntrack | wc -l

# 한도
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_tcp_timeout_established
```

### 7.2 운영 영향

| 현상 | 원인 | 모델 설명 |
| --- | --- | --- |
| `nf_conntrack: table full, dropping packet` | conntrack table 한도 초과 | high-concurrency 노드 / NAT gateway 에서 흔함 |
| TCP connection 이 갑자기 끊김 | conntrack TCP timeout (default 5 day) 도달 또는 INVALID | tcp keepalive / 짧은 timeout 설정 |
| DNAT 후 응답이 잘못된 경로 | NAT 한쪽만 적용 (asymmetric routing) | conntrack 가 양방향 entry 자동 생성 |
| Docker / k8s 노드의 packet drop | conntrack max 부족 | `nf_conntrack_max` 증가 |

---

## 8. DevOps 도구의 netfilter 사용

| 도구 | 사용 방식 |
| --- | --- |
| **Docker** | bridge network + iptables 의 nat (MASQUERADE) + filter (DOCKER chain) 자동 생성. `--iptables=false` 로 끌 수 있으나 default ON. |
| **kube-proxy (iptables mode)** | Service 마다 nat PREROUTING + KUBE-SERVICES / KUBE-SVC-* / KUBE-SEP-* chain 생성. Pod endpoint 로 DNAT. |
| **kube-proxy (ipvs mode)** | iptables 대신 IPVS (L4 LB) 사용. 룰 수 ≥ 1000 service 시 권장. |
| **kube-proxy (Cilium replacement)** | iptables / IPVS 우회. eBPF 가 직접 packet 처리. [[../../../devops/networking-ops/cni-deep]]. |
| **firewalld** | iptables / nftables 의 high-level wrapper. zone / service 추상화. RHEL / Fedora 표준. |
| **ufw** | iptables / nftables 의 high-level wrapper. Ubuntu 표준. |
| **fail2ban** | log 분석 후 iptables 룰 동적 추가 (IP 차단). |
| **k8s NetworkPolicy** | CNI 가 구현. Calico / Cilium 이 iptables 또는 eBPF 로 enforcement. |
| **WireGuard** | netfilter 와 무관 (커널 모듈). but 같이 사용 시 routing / NAT 룰 필요. |

---

## 9. 디버깅 절차

### 9.1 룰 확인

```bash
# 모든 table 의 모든 chain
iptables-save
nft list ruleset

# 통과 packet 수 + byte 수 (counter)
iptables -L -n -v
nft list ruleset -a

# Docker 의 추가 룰
iptables -t nat -L DOCKER -n -v
iptables -L DOCKER-USER -n -v       # 사용자 추가 룰 위치

# kube-proxy iptables 룰
iptables-save | grep -E "KUBE|svc/"
```

### 9.2 처리 추적

```bash
# packet 별 어디서 어떻게 처리되는지
# 방법 1: TRACE target (table=raw)
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
dmesg -w   # 또는 journalctl -k -f

# 방법 2: nft 의 nftrace
nft monitor trace

# 방법 3: bpftrace
bpftrace -e 'kprobe:nf_hook_slow { printf("hook=%d\n", arg1); }'
```

### 9.3 conntrack 확인

```bash
conntrack -L | grep <src_ip>
conntrack -S            # insert_failed / drop / invalid / search_restart 확인
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 추적 |
| --- | --- | --- |
| Docker 컨테이너가 외부 접근 못함 | `iptables -P FORWARD DROP` + DOCKER-USER 룰 부족 | `iptables -L FORWARD -n -v` 의 counter |
| k8s Service 가 routing 안 됨 | kube-proxy 가 죽음 / iptables 룰 누락 | `iptables-save | grep KUBE-SVC` |
| 룰 수 폭주로 응답 느림 | iptables O(N) 평가 — 1000+ Service | ipvs / cilium 으로 교체 |
| `nf_conntrack: table full` | conntrack max 부족 + 짧은 timeout 미설정 | §7.2 |
| reboot 후 룰 사라짐 | iptables-save / iptables-persistent 미설정 | `/etc/iptables/rules.v4` 저장 |
| firewalld 와 iptables 직접 수정 충돌 | firewalld 가 reload 시 직접 룰 제거 | firewalld zone / direct rule 사용 |
| MASQUERADE 후 source IP 보존 안 됨 | NAT 의 본질 — log / audit 영향 | SNAT 명시 또는 X-Forwarded-For |
| asymmetric routing 의 conntrack INVALID | 양쪽 packet 이 다른 NIC 통과 | routing fix 또는 `rp_filter` 조정 |

---

## 11. 참고

- [[tools|↑ tools]]
- [[../network|↑ network hub]]
- [[../ip/nat]] — NAT 의 의미
- [[container-networking]] — Docker / k8s 가 만드는 packet 흐름
- [[tcp-socket-ops-pitfalls]] — conntrack timeout 의 TCP 영향
- [[../../../devops/networking-ops/network-policy]] — k8s NetworkPolicy 의 enforcement
- [[../../../devops/networking-ops/cni-deep]] — CNI 별 packet 처리 (iptables / IPVS / eBPF)
- [[../../../devops/linux/linux-mental-models-for-devops]] — Linux 커널 모델
- man iptables, man nft
- netfilter docs — https://www.netfilter.org/documentation/
