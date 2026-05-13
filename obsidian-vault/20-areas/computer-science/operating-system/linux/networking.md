---
title: "Linux 네트워크 운영 — ip / ss / iptables / nft"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:40:00+09:00
tags:
  - operating-system
  - linux
  - network
---

# Linux 네트워크 운영

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | iproute2 / netfilter / 진단 |

**[[linux|↑ Linux hub]]**

---

개념은 → [[../../network/network]]
여기는 **Linux 실무** 만.

---

## 1. ip (iproute2 — 표준)

```bash
ip a                                  # 모든 인터페이스 / IP
ip a show eth0
ip link show
ip link set eth0 up / down
ip link set eth0 mtu 9000

ip addr add 10.0.0.10/24 dev eth0
ip addr del 10.0.0.10/24 dev eth0

ip r                                  # 라우팅
ip route add 10.1.0.0/16 via 10.0.0.1
ip route del default
ip route get 8.8.8.8                   # 어떤 인터페이스로

ip neighbor                            # ARP
ip neighbor add 10.0.0.5 lladdr 00:11:22:33:44:55 dev eth0
ip neighbor del 10.0.0.5 dev eth0

ip rule                                # policy routing
```

옛: `ifconfig`, `route`, `arp` — deprecated. `ip` 만 사용.

---

## 2. 영구 설정

### 2.1 netplan (Ubuntu 18+)

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.0.0.10/24]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
```

### 2.2 NetworkManager

```bash
nmcli device status
nmcli con show
nmcli con add type ethernet con-name eth0 ifname eth0 ip4 10.0.0.10/24 gw4 10.0.0.1
nmcli con mod eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con up eth0
```

### 2.3 systemd-networkd

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=10.0.0.10/24
Gateway=10.0.0.1
DNS=8.8.8.8
```

### 2.4 /etc/network/interfaces (옛 Debian)

```
auto eth0
iface eth0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
```

---

## 3. ss (sockets statistics)

`netstat` 의 후계.

```bash
ss -tln                                # TCP listen
ss -tnp                                 # TCP + process
ss -uln                                 # UDP listen
ss -tunap                               # 모두 + 모든 상태
ss -s                                    # summary
ss -i                                    # internal info (cwnd, rtt)
ss -o state established
ss -t state established '( dport = :443 or sport = :443 )'
ss -x                                    # Unix
```

### 3.1 자주 쓰는

```bash
ss -tlnp | grep nginx
ss -tnp '( sport = :80 )'
ss state syn-sent                        # SYN_SENT
ss -tnp '( dst 1.2.3.4 )'
```

---

## 4. 진단

```bash
ping host
ping -c 4 8.8.8.8

# 경로
traceroute 8.8.8.8
mtr 8.8.8.8                              # 가장 좋음 — 지속 모니터링

# DNS
dig example.com
dig +short example.com
dig @8.8.8.8 example.com
dig example.com MX
nslookup example.com
host example.com
resolvectl query example.com             # systemd-resolved

# 포트 / 연결
nc -zv host 22                           # 포트 열림?
nc -lv 8080                               # listen
ncat / socat                             # 더 강력
telnet host 80                            # 옛

# 시간 측정
curl -o /dev/null -s -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://...
```

---

## 5. tcpdump / Wireshark

```bash
sudo tcpdump -i eth0 -nn
sudo tcpdump -i eth0 -nn 'tcp port 80'
sudo tcpdump -i any 'host 10.0.0.5'
sudo tcpdump -i any 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
sudo tcpdump -w capture.pcap port 443
sudo tcpdump -r capture.pcap
sudo tcpdump -X -nn 'port 80' -c 10        # hex dump

# Wireshark / tshark
wireshark capture.pcap
tshark -i eth0 -Y 'http'
```

자세히 → [[../../network/tools/tools]]

---

## 6. iptables (옛, 여전히 사용)

```bash
# 보기
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v

# 규칙
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -j DROP

# 저장
sudo iptables-save > /etc/iptables/rules.v4
```

### 6.1 chain

```
INPUT       — 들어오는
OUTPUT      — 나가는
FORWARD     — passing
PREROUTING  — 들어와 routing 전 (NAT)
POSTROUTING — 나가기 직전 (NAT)
```

### 6.2 table

```
filter (기본) — 허용 / 거부
nat            — SNAT / DNAT
mangle         — packet modify
raw            — connection tracking 우회
```

---

## 7. nftables (현대)

iptables 의 후계. 모든 protocol / table 하나의 도구.

```bash
sudo nft list ruleset
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0 \; }'
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input drop
```

문법:
```
table inet filter {
    chain input {
        type filter hook input priority 0;
        ct state established,related accept
        tcp dport {22, 80, 443} accept
        iif lo accept
        drop
    }
}
```

---

## 8. firewalld / ufw (front-end)

### 8.1 ufw (Ubuntu)

```bash
sudo ufw enable
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow from 10.0.0.0/8
sudo ufw status verbose
sudo ufw delete allow 80
```

### 8.2 firewalld (RHEL)

```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-services
sudo firewall-cmd --set-default-zone=public
```

zone 개념 — interface 단위 정책.

---

## 9. NAT / 포트 포워드

```bash
# iptables
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:80
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
echo 1 > /proc/sys/net/ipv4/ip_forward

# nftables
nft add rule nat prerouting tcp dport 80 dnat to 10.0.0.5:80
```

---

## 10. bridge / VLAN / bond

### 10.1 bridge

```bash
sudo ip link add br0 type bridge
sudo ip link set eth0 master br0
sudo ip link set br0 up
```

container / VM 의 가상 스위치.

### 10.2 VLAN

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
```

### 10.3 bond (link aggregation)

```bash
modprobe bonding
echo +bond0 > /sys/class/net/bonding_masters
echo 4 > /sys/class/net/bond0/bonding/mode      # 802.3ad
echo +eth0 > /sys/class/net/bond0/bonding/slaves
echo +eth1 > /sys/class/net/bond0/bonding/slaves
```

---

## 11. /etc/hosts / /etc/resolv.conf

```bash
# /etc/hosts
127.0.0.1   localhost
10.0.0.5    db.local

# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com
```

systemd-resolved 가 /etc/resolv.conf 를 동적 관리.

---

## 12. sysctl (network)

```bash
# /etc/sysctl.d/99-net.conf
net.ipv4.ip_forward = 1
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_tw_reuse = 1
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_congestion_control = bbr
```

```bash
sudo sysctl --system                       # apply
sudo sysctl net.ipv4.ip_forward=1          # 임시
```

자세히 → [[tuning]]

---

## 13. Container / K8s network

자세히 → [[../virtualization/container]]

namespace `net` + veth pair + bridge / VXLAN.
CNI plugin: Calico, Flannel, Cilium (eBPF).

---

## 14. 네트워크 namespace

```bash
sudo ip netns add red
sudo ip netns exec red ip a
sudo ip link add veth-red type veth peer name veth-red-host
sudo ip link set veth-red netns red
sudo ip netns exec red ip addr add 10.99.0.1/24 dev veth-red
sudo ip netns exec red ip link set veth-red up
sudo ip netns exec red ip link set lo up
```

자세히 → [[../virtualization/namespace#7-net-namespace]]

---

## 15. 함정

### 15.1 iptables 와 firewalld 동시
충돌. 하나만.

### 15.2 ufw 의 default deny
ssh 가 막혀 lockout — `allow 22` 먼저.

### 15.3 systemd-resolved 의 /etc/resolv.conf
symlink. 직접 편집해도 덮어씌워짐.

### 15.4 NetworkManager + netplan 충돌
서버 = systemd-networkd. 데스크탑 = NetworkManager.

### 15.5 MTU mismatch
VPN / VLAN 후 1500 → fragmentation. mtu 조정.

### 15.6 hostname / DNS reverse
PTR 미설정 — log 가 IP 만.

### 15.7 cloud network 의 source/dest check
AWS 에서 NAT instance 시 비활성 필요.

### 15.8 IPv6 의 영향
disabled 가정 — 응용이 IPv6 first → 느림. resolv.conf, sysctl.

---

## 16. 학습 자료

- **iproute2** docs
- **The Linux Networking Cookbook**
- **Linux Network Administrator's Guide**
- **nftables wiki**

---

## 17. 관련

- [[../../network/network]] — 개념
- [[monitoring]] — 통계
- [[tuning]] — sysctl
- [[linux]] — Linux hub
