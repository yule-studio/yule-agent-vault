---
title: "ss / netstat / lsof — 소켓 / 연결 분석"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:30:00+09:00
tags:
  - network
  - tools
  - ss
  - netstat
  - lsof
---

# ss / netstat / lsof — 소켓 / 연결 분석

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TCP state / port / pid |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. ss — Socket Statistics (모던)

### 정의
- iproute2 의 일부
- netstat 의 모던 대체 — 더 빠름

### 기본
```bash
ss                  # 모든 socket
ss -t               # TCP only
ss -u               # UDP only
ss -l               # listening
ss -tlnp            # TCP listen + numeric port + process
ss -tan             # TCP all + numeric
```

### 옵션
| | 의미 |
| --- | --- |
| `-t` | TCP |
| `-u` | UDP |
| `-x` | Unix socket |
| `-l` | Listening |
| `-a` | All (listen + connected) |
| `-n` | Numeric (DNS X) |
| `-p` | Process (root) |
| `-s` | Summary 통계 |
| `-i` | Internal info (cwnd / rtt) |
| `-o` | Timer (retrans / keepalive) |
| `-m` | Memory |

### 결과
```
State      Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
ESTAB      0       0       1.2.3.4:443          5.6.7.8:50001
LISTEN     0       128     0.0.0.0:80           0.0.0.0:*          users:(("nginx",pid=123,fd=6))
```

### 필터
```bash
ss -t state established
ss -t state time-wait
ss -t '( dport = :443 or dport = :80 )'
ss -t '( sport = :22 )'
ss -t dst 1.2.3.4
ss -t dst 1.2.3.4/16
```

### TCP 상세 (-i)
```bash
ss -tin
# cubic wscale:7,7 rto:204 rtt:0.5/0.25 ato:40 mss:1448 cwnd:10
```

| | 의미 |
| --- | --- |
| cubic | Congestion control |
| rtt | RTT in ms |
| cwnd | Congestion window |
| mss | Max segment size |
| rto | Retransmission timeout |

---

## 2. netstat — 옛 / 호환

### 기본
```bash
netstat -tlnp           # 같은 ss -tlnp
netstat -anp
netstat -s              # 통계
netstat -r              # routing table
netstat -i              # interface
```

### 모던 Linux — ss 권장
- netstat — /proc 읽기 (느림)
- ss — netlink (빠름)

### macOS — netstat 만
```bash
netstat -an | grep LISTEN
netstat -nr             # routing
netstat -i              # interface
```

---

## 3. lsof — Open Files

### 정의
- 모든 열린 파일 / 소켓 — 어떤 process

### 네트워크
```bash
sudo lsof -i                      # 모든 network
sudo lsof -i :8080                # 특정 port
sudo lsof -i TCP:80
sudo lsof -i UDP
sudo lsof -i @1.2.3.4             # 특정 IP
sudo lsof -p 1234                 # 특정 process
sudo lsof -i -P -n                # numeric (빠름)
```

### 결과
```
COMMAND   PID   USER   FD   TYPE  DEVICE  NODE NAME
nginx    1234   root   6u  IPv4   12345   TCP  *:80 (LISTEN)
nginx    1234   root   7u  IPv4   12346   TCP  1.2.3.4:80->5.6.7.8:50001 (ESTABLISHED)
```

### 사용
- "이 port 사용하는 프로세스?"
- File descriptor leak
- 잠긴 파일 추적

---

## 4. TCP 상태 (state)

| State | 의미 |
| --- | --- |
| LISTEN | 수신 대기 |
| SYN_SENT | SYN 보냄 (client) |
| SYN_RECV | SYN 받음 (server) |
| ESTABLISHED | 연결됨 |
| FIN_WAIT_1 | 종료 SYN 보냄 (active close) |
| FIN_WAIT_2 | ACK 받음, FIN 대기 |
| TIME_WAIT | 마지막 ACK 후 2MSL 대기 (active closer) |
| CLOSE_WAIT | FIN 받음 (passive close) — close() 대기 |
| LAST_ACK | passive close 의 마지막 |
| CLOSED | 종료 |

자세히 → [[../tcp/tcp-state-machine]]

### CLOSE_WAIT 누적
- 서버가 close() 안 함 → file descriptor leak
- 코드 버그

### TIME_WAIT 누적
- 짧은 connection 매우 많음
- net.ipv4.tcp_tw_reuse 또는 longer-lived connection

---

## 5. 흔한 시나리오

### 5.1 누가 이 port 쓰나?
```bash
sudo lsof -i :8080
sudo ss -tlnp | grep :8080
sudo fuser 8080/tcp
```

### 5.2 ESTABLISHED 수
```bash
ss -tan state established | wc -l
```

### 5.3 TIME_WAIT 수
```bash
ss -tan state time-wait | wc -l
# 큰 경우 — 짧은 connection 많음
```

### 5.4 어디 connection 가 가장 많나?
```bash
ss -tan | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

### 5.5 한 process 의 connection
```bash
sudo lsof -p 1234 -i
```

### 5.6 Open file descriptor
```bash
ls /proc/1234/fd | wc -l
ulimit -n
```

---

## 6. iftop — 실시간 대역

### 정의
- 인터페이스 별 실시간 traffic
- bandwidth top

```bash
sudo iftop -i en0
sudo iftop -P                   # port 표시
sudo iftop -n                   # no DNS
```

---

## 7. nethogs — Process 별 대역

```bash
sudo nethogs
# 프로세스 — sent/received bandwidth
```

---

## 8. bmon — 대역 모니터

```bash
bmon                            # 인터페이스 별 그래프
```

---

## 9. /proc/net/ — Linux internals

```bash
cat /proc/net/tcp               # TCP socket
cat /proc/net/udp
cat /proc/net/dev               # interface stat
cat /proc/net/snmp              # protocol stat
cat /proc/net/route             # routing
```

→ ss / netstat 의 원본.

---

## 10. ifconfig / ip — 인터페이스

### 옛 — ifconfig
```bash
ifconfig
ifconfig en0
sudo ifconfig en0 up
sudo ifconfig en0 192.168.1.10 netmask 255.255.255.0
```

### 모던 — ip
```bash
ip addr show
ip addr add 192.168.1.10/24 dev eth0
ip link show
ip route show
ip neigh show                   # ARP 캐시
```

---

## 11. arp / ip neigh

### ARP 캐시
```bash
arp -a                          # 옛
ip neigh show                   # 모던
```

### 사용
- 같은 망의 IP ↔ MAC 매핑
- 캐시 청소
```bash
sudo arp -d 192.168.1.10
sudo ip neigh del 192.168.1.10 dev eth0
```

---

## 12. 함정

### 함정 1 — sudo 없으면 process 안 보임
ss / netstat / lsof — root 권한 필요 (다른 사용자 process).

### 함정 2 — netstat 의 성능
큰 시스템 — /proc 읽기 느림. ss 권장.

### 함정 3 — CLOSE_WAIT 누적
서버 코드 — close() 누락. 재시작 임시 — 코드 fix 필수.

### 함정 4 — TIME_WAIT 의 의미
정상 동작. 너무 많으면 — connection re-use 검토.

### 함정 5 — ESTABLISHED 의 stale
keepalive 없으면 — 서버 죽었어도 client 모름. 운영 — keepalive 설정.

### 함정 6 — Container 안의 ss
nsenter 또는 host 의 lsof 가 진짜 상태.

---

## 13. 학습 자료

- `man ss`, `man netstat`, `man lsof`
- "Linux Performance" (Gregg)
- "TCP/IP Illustrated"

---

## 14. 관련

- [[tools]] — Hub
- [[../tcp/tcp-state-machine]] — state 의 의미
- [[tcpdump-wireshark]] — 패킷 단위
