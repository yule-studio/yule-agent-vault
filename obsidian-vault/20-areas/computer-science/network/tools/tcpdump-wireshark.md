---
title: "tcpdump / Wireshark — 패킷 캡처 / 분석"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:05:00+09:00
tags:
  - network
  - tools
  - tcpdump
  - wireshark
---

# tcpdump / Wireshark — 패킷 캡처 / 분석

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | BPF / filter / display filter |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. tcpdump

### 기본
```bash
sudo tcpdump -i en0
sudo tcpdump -i any                 # 모든 interface
sudo tcpdump -D                     # interface 목록
```

### 옵션
```
-i <interface>   인터페이스
-n               IP 숫자 (DNS X)
-nn              port 도 숫자
-v / -vv / -vvv  자세히
-X               hex + ASCII
-c <n>           n 패킷만
-w <file.pcap>   파일로 저장
-r <file.pcap>   파일 읽기
-s 0             전체 패킷 (snaplen)
-Z root          root 권한 유지
```

### BPF 필터 — 가장 흔한

```bash
# host
sudo tcpdump -i en0 host 1.2.3.4
sudo tcpdump src host 1.2.3.4
sudo tcpdump dst host 1.2.3.4
sudo tcpdump not host 1.2.3.4

# port
sudo tcpdump port 443
sudo tcpdump src port 443
sudo tcpdump 'dst port 80 or dst port 443'
sudo tcpdump portrange 8000-9000

# 프로토콜
sudo tcpdump icmp
sudo tcpdump tcp
sudo tcpdump udp
sudo tcpdump arp

# 결합
sudo tcpdump 'tcp port 443 and host example.com'
sudo tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'    # SYN/FIN
sudo tcpdump 'tcp[tcpflags] = tcp-syn'                    # SYN only

# 네트워크
sudo tcpdump net 10.0.0.0/24
```

### 출력 해석
```
12:00:00.123456 IP 1.2.3.4.49152 > 5.6.7.8.443: Flags [S], seq 123, win 65535
                ↑ 시간          ↑ src            ↑ dst        ↑ flags        ↑ seq
```

### 파일 저장 / 분석
```bash
# 캡처
sudo tcpdump -i en0 -w capture.pcap host example.com -c 1000

# 읽기 (tcpdump)
tcpdump -r capture.pcap

# 읽기 + 필터
tcpdump -r capture.pcap 'tcp port 443'

# Wireshark / tshark 로 분석 (더 강력)
```

---

## 2. tshark — Wireshark CLI

```bash
# 기본
sudo tshark -i en0

# Display filter (BPF 보다 풍부)
sudo tshark -Y "http.host == example.com"
sudo tshark -Y "tls.handshake.type == 1"      # ClientHello
sudo tshark -Y "dns.qry.name contains google"
sudo tshark -Y "tcp.analysis.retransmission"

# 통계
sudo tshark -z conv,tcp -r capture.pcap        # connection 별
sudo tshark -z io,stat,1 -r capture.pcap       # 1 초 단위
sudo tshark -z http,tree -r capture.pcap       # HTTP 트리
```

---

## 3. Wireshark — GUI

### Display filter
```
ip.addr == 1.2.3.4
tcp.port == 443
http.host == "example.com"
http.request.method == "POST"
tls.handshake.type == 1
dns.qry.name == "example.com"

# 결합
ip.src == 1.2.3.4 and tcp.flags.syn == 1

# 분석
tcp.analysis.retransmission
tcp.analysis.duplicate_ack
tcp.analysis.zero_window
tcp.analysis.window_full
```

### 분석 메뉴
- Statistics → I/O Graph (시간 별 패킷)
- Statistics → Conversations (5-tuple 별)
- Statistics → Flow Graph (시간 순)
- Statistics → TCP Stream Graph
- Analyze → Follow → TCP Stream (한 connection 만)
- Analyze → Expert Information (이상 자동)

### TLS 복호화
```
Edit → Preferences → Protocols → TLS
(Pre)-Master-Secret log filename: /path/to/sslkeylogfile
```

### SSLKEYLOGFILE
```bash
# Chrome / Firefox 가 자동 로깅
export SSLKEYLOGFILE=/tmp/sslkeys.log
chrome &
```

→ Wireshark 가 자동 복호화.

---

## 4. 흔한 시나리오

### 4.1 TCP 3-way handshake
```bash
sudo tcpdump -i en0 -nn 'host example.com and tcp port 443'
# SYN → SYN-ACK → ACK
```

### 4.2 TLS handshake
```
Display filter: tls.handshake
Follow: SNI / cipher / cert
```

### 4.3 Retransmission
```
Display filter: tcp.analysis.retransmission
```

→ 네트워크 문제 / 패킷 손실.

### 4.4 DNS query / response
```bash
sudo tcpdump -i en0 -nn 'udp port 53'
# 또는 Wireshark: dns
```

### 4.5 HTTP 분석
```
Display filter: http
Statistics → HTTP → Requests
```

### 4.6 ARP
```bash
sudo tcpdump -i en0 arp
```

---

## 5. tcpdump 가 못 보는 것

### TLS / SSL
- 헤더만 (handshake 후 — 암호화)
- SSLKEYLOGFILE 로 복호화 (Wireshark)

### Loopback (lo)
- `-i lo` — local 트래픽
- macOS — `-i lo0`

### Encrypted tunnel (VPN)
- VPN 안 — 평문, 밖 — 암호화

---

## 6. nfdump / flow

### NetFlow / sFlow / IPFIX
- 패킷 단위 X — flow (5-tuple aggregated)
- 라우터 / 스위치 가 export

### nfdump
```bash
nfdump -r flow_file -o "fmt:%ts %sap → %dap %byt"
nfdump -r flow_file -s ip/bytes        # top IP by bytes
```

---

## 7. mitmproxy — HTTP 가로채기

```bash
mitmproxy --listen-port 8080
mitmweb --listen-port 8080          # web UI
mitmdump -r capture.flow            # 분석

# 클라이언트 — proxy 설정
# CA cert 설치 (mitm.it)
```

### 사용
- API 디버깅 (request / response 보기)
- 변조 (replay 후 modify)
- 스크립트 (Python)

---

## 8. 함정

### 함정 1 — Loopback 누락
`-i any` 도 loopback 누락 가능. `-i lo0` (macOS) / `-i lo` (Linux) 명시.

### 함정 2 — sudo 필요
패킷 캡처 — root / pcap 권한. 그룹 (Linux — `setcap cap_net_raw=ep`).

### 함정 3 — snaplen 작음
옛 기본 96 byte — payload 잘림. `-s 0` (전체).

### 함정 4 — 큰 파일
계속 캡처 — disk full. `-G 60 -W 10` (60s 마다 rotate, 10 파일).

### 함정 5 — TLS 복호화의 보안
SSLKEYLOGFILE — 개발 환경만. 운영 X.

### 함정 6 — 정확한 timestamp
PC clock drift — 패킷 timestamp 부정확. 네트워크 측 — NTP 동기.

### 함정 7 — Filter 의 expression
BPF (capture) vs Display filter (Wireshark) 다름.

---

## 9. 학습 자료

- `man tcpdump`
- "Practical Packet Analysis" (Sanders)
- Wireshark.org docs
- "Wireshark Network Analysis" (Chappell)

---

## 10. 관련

- [[tools]] — Hub
- [[../tcp/tcp]] — flag 의 의미
- [[../tls-ssl/tls-ssl]] — handshake 분석
- [[curl-httpie]] — HTTP 디버깅
