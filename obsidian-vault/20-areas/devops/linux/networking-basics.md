---
title: "Networking basics — ip / ss / curl / dig"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:48:00+09:00
tags: [devops, linux, networking]
---

# Networking basics — ip / ss / curl / dig

**[[linux|↑ linux]]**

---

## 1. interface 확인

```bash
ip a                          # 모든 interface
ip addr show eth0
ip link                       # link 상태 (UP/DOWN, MTU)
ip route                      # routing table
ip -s link show eth0          # stat (RX/TX bytes)

# legacy (deprecated)
ifconfig
route -n
```

---

## 2. socket / port

```bash
ss -tnl                       # TCP listening
ss -tun                       # TCP + UDP non-listen
ss -tnp                       # process 포함
ss -tn '( dst :443 )'         # filter
ss -s                          # 통계

# 흔한
ss -tnlp | grep :8080         # 8080 listen 누구?
ss -tn state established      # 연결된 socket

# legacy
netstat -tlnp
netstat -an
```

---

## 3. DNS

```bash
dig example.com               # A
dig example.com AAAA           # IPv6
dig example.com MX             # mail
dig example.com TXT            # SPF / DKIM
dig +short example.com         # 짧게
dig @8.8.8.8 example.com       # 특정 resolver

nslookup example.com           # 간단한 도구
host example.com               # 간단한 도구

# 역방향
dig -x 8.8.8.8                 # PTR
```

→ `/etc/resolv.conf` 가 resolver 설정.  
→ `systemd-resolved` 사용 시 `resolvectl status`.

---

## 4. /etc/hosts

```
# /etc/hosts
127.0.0.1   localhost
::1         localhost
10.0.0.5    db.internal
10.0.0.6    redis.internal
```

→ DNS 보다 우선 (보통). `/etc/nsswitch.conf` 에서 순서 결정.

---

## 5. curl (HTTP 디버깅)

```bash
curl -v https://api.example.com           # verbose (header 포함)
curl -I https://api.example.com           # HEAD only
curl -L https://api.example.com           # redirect follow
curl -o file.tar https://...              # 다운로드
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"a"}' https://api/users
curl -u alice:secret https://api          # basic auth
curl -H "Authorization: Bearer $TOKEN"
curl --resolve api.example.com:443:10.0.0.5 https://api.example.com   # DNS override
curl -w "@curl-format.txt" -o /dev/null -s https://api      # timing

# curl-format.txt
#    time_namelookup:  %{time_namelookup}\n
#       time_connect:  %{time_connect}\n
#    time_appconnect:  %{time_appconnect}\n
#   time_pretransfer:  %{time_pretransfer}\n
#      time_redirect:  %{time_redirect}\n
# time_starttransfer:  %{time_starttransfer}\n
#                    ----------\n
#         time_total:  %{time_total}\n
```

---

## 6. nc / ncat (port 테스트)

```bash
nc -zv host 80                # port 열려있나?
nc -l 9090                    # listen (test server)

# host 사이 채팅
# server: nc -l 9090
# client: nc server 9090

# port 스캔 (간단)
nc -zv host 80 443 8080
```

---

## 7. tcpdump (패킷)

```bash
tcpdump -i eth0 -nn               # 그냥
tcpdump -i eth0 port 80
tcpdump -i eth0 host 10.0.0.5
tcpdump -i eth0 -A -s 0 port 8080 # ASCII payload
tcpdump -w out.pcap -i eth0       # file 로 저장 (wireshark 에서 분석)
tcpdump -r out.pcap

# 흔한 케이스
tcpdump -i any port 443 -nn -A
```

→ root 권한 필요. 디버깅 막바지 도구.

---

## 8. traceroute / mtr

```bash
traceroute -n example.com         # hop 별
mtr example.com                   # 실시간 + loss
mtr --report --report-cycles 50 example.com
```

→ network 어디서 막히는지.

---

## 9. iptables / nftables (방화벽)

```bash
# iptables (legacy)
iptables -L -n -v                 # 현재 rule
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -j DROP
iptables -F                        # flush

# nftables (현재 표준)
nft list ruleset
nft add rule inet filter input tcp dport 22 accept

# 더 쉬운 wrapper
ufw allow 22/tcp                  # Ubuntu
firewall-cmd --add-port=22/tcp    # RHEL
```

---

## 10. MTU / fragmentation

```bash
ip link show eth0 | grep mtu      # 보통 1500
ping -M do -s 1472 host           # 1472+28 = 1500 (확인)
```

→ VPN / tunnel 환경에서 MTU 작게 조정 필요할 수 있음.

---

## 11. /etc/services / well-known port

```
ssh        22
http       80
https      443
postgres   5432
mysql      3306
redis      6379
kafka      9092
mongo      27017
```

---

## 12. 흔한 진단 흐름

```
1. DNS?           dig host
2. routing?       traceroute / mtr
3. firewall?      nc -zv host port
4. service 동작?  ss -tnl on server
5. HTTP 응답?     curl -v
6. 패킷?          tcpdump
```

---

## 13. 함정

1. **ping 막힘** — ICMP block. nc / curl 로 검증.
2. **DNS cache** — `systemd-resolve --flush-caches` / `resolvectl flush-caches`.
3. **iptables 규칙 순서** — 위에서부터 매칭 → DROP 이 먼저면 ACCEPT 안 됨.
4. **conntrack overflow** — 많은 connection 시 `net.netfilter.nf_conntrack_max`.
5. **/etc/hosts override** — DNS 결과 와 다름 → 디버깅 시간 낭비.

---

## 14. 관련

- [[linux|↑ linux]]
- [[ssh]]
- [[../networking-ops/networking-ops|↗ networking-ops]]
- [[../nginx/nginx|↗ nginx]]
