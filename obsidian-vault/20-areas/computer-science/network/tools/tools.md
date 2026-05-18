---
title: "네트워크 도구 — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:00:00+09:00
tags:
  - network
  - tools
---

# 네트워크 도구 — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 도구 인벤토리 + 시나리오 매핑 |
| v.1.1.0 | 2026-05-18 | engineering-agent/tech-lead | iptables/netfilter 추가 (Linux packet filter — Docker / kube-proxy / firewall 의 기반) |

**[[../network|↑ network hub]]**

---

## 1. 한 줄 정의

네트워크 디버깅 / 측정 / 분석의 **도구 인벤토리**. 시나리오 별 적합한 도구 매핑.

---

## 2. 시나리오 별 매핑

| 시나리오 | 도구 |
| --- | --- |
| DNS 조회 | dig, nslookup, host |
| HTTP 디버깅 | curl, httpie, Postman |
| 연결 확인 | ping, mtr, traceroute |
| 패킷 캡처 | tcpdump, Wireshark |
| 포트 스캔 | nmap |
| 대역 측정 | iperf3, speedtest |
| TCP 분석 | ss, netstat |
| TLS 검증 | openssl s_client, sslyze |
| Latency 추적 | mtr, pathping |
| gRPC 디버깅 | grpcurl, ghz |
| Load test | wrk, hey, k6, Locust |
| RTT 측정 | hping, ping |
| packet filter / NAT / firewall | iptables, nftables, conntrack |

---

## 3. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[tcpdump-wireshark]] | 패킷 캡처 |
| [[curl-httpie]] | HTTP 클라이언트 |
| [[nmap-port-scan]] | 포트 스캔 |
| [[iperf-load-testing]] | 대역 / 부하 |
| [[ping-traceroute-mtr]] | 경로 / 지연 |
| [[ss-netstat-tcp-tools]] | 소켓 / TCP 상태 |
| [[iptables-netfilter]] | Linux packet filter / NAT / conntrack — Docker / kube-proxy / firewalld 의 기반 |

---

## 4. DNS (기존 노트)

자세히 → [[../dns/dns-tools]]

- dig / nslookup / host / kdig / dnsperf

---

## 5. 한눈 — 빠른 명령

### 연결 확인
```bash
ping example.com
ping6 example.com
mtr example.com               # ping + traceroute
traceroute example.com
```

### DNS
```bash
dig example.com
dig +short example.com
dig example.com MX
nslookup example.com
```

### HTTP
```bash
curl -v https://example.com
curl -I https://example.com         # headers only
curl --resolve example.com:443:1.2.3.4 https://example.com
http https://example.com            # httpie
```

### TLS
```bash
openssl s_client -connect example.com:443 -servername example.com
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -text
```

### 패킷
```bash
sudo tcpdump -i en0 -nn port 443
sudo tcpdump -i en0 -w capture.pcap host example.com
```

### 포트 / 소켓
```bash
sudo lsof -i :8080
sudo ss -tlnp
sudo netstat -tlnp
```

### 부하
```bash
wrk -t10 -c100 -d30s https://example.com
hey -n 1000 -c 10 https://example.com
```

---

## 6. macOS vs Linux 의 차이

| | macOS | Linux |
| --- | --- | --- |
| ping | BSD | iputils |
| traceroute | UDP | UDP / ICMP |
| netstat | BSD-style | iproute2 권장 (ss) |
| ifconfig | 사장 | ip (iproute2) |

### macOS — 추가 도구
- `dscacheutil -flushcache`
- `networksetup`
- `airport` (WiFi)

### Linux — 모던
- `ip` (iproute2) — ifconfig 대체
- `ss` — netstat 대체
- `nmcli` — NetworkManager

---

## 7. 도구 — 클래식 vs 모던

| 옛 | 모던 |
| --- | --- |
| ifconfig | ip |
| netstat | ss |
| route | ip route |
| arp | ip neigh |
| nslookup | dig / kdig |
| tcpdump | tshark / mitmproxy |
| ab | wrk / hey / k6 |
| telnet | nc / openssl s_client |

---

## 8. 학습 자료

- "TCP/IP Illustrated" — 도구 예제
- "Network Warrior" (Donahue)
- 각 도구 man page

---

## 9. 관련

- [[../dns/dns-tools]] — DNS 도구
- 세부 노트들 (tcpdump-wireshark / curl-httpie / nmap / iperf / mtr / ss)
