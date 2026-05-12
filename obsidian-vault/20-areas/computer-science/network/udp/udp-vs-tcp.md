---
title: "UDP vs TCP — 비교 가이드"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:20:00+09:00
tags:
  - network
  - udp
  - tcp
---

# UDP vs TCP — 비교 가이드

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 표 비교 + 시나리오별 선택 |

**[[udp|↑ UDP]]** · **[[../tcp/tcp|↗ TCP]]**

---

## 1. 핵심 차이 표

| 측면 | TCP | UDP |
| --- | --- | --- |
| 연결 | 연결 지향 | 비연결 |
| 신뢰성 | 보장 (재전송) | 없음 |
| 순서 | 보장 (Seq#) | 없음 |
| 흐름 제어 | rwnd | 없음 |
| 혼잡 제어 | cwnd (CUBIC/BBR) | 없음 |
| 헤더 | 20-60 byte | 8 byte |
| 메시지 경계 | 보존 X (스트림) | 보존 (1 send = 1 recv) |
| 멀티캐스트 | ❌ | ✅ |
| 핸드셰이크 | 3-way (1 RTT) | 없음 |
| HoL Blocking | 있음 | 없음 |
| Latency | 일정 | 변동 |
| Throughput | 안정 | 폭주 가능 |
| 표준 | RFC 793 / 9293 | RFC 768 |

---

## 2. 시나리오별 선택

### 2.1 TCP 가 답
- HTTP / HTTPS (큰 파일, JSON API)
- SSH / Telnet
- FTP / SFTP
- 이메일 (SMTP / IMAP / POP3)
- 데이터베이스 (MySQL, PostgreSQL)
- Git / svn / Mercurial
- Redis (텍스트 또는 RESP 프로토콜)

### 2.2 UDP 가 답
- DNS (작은 쿼리, TCP fallback)
- NTP (시간 정확도 중요)
- DHCP (broadcast 필요)
- VoIP / 화상 회의 (RTP / WebRTC)
- 게임 (실시간)
- 라이브 스트리밍 (저지연)
- Monitoring (SNMP, Syslog)
- IoT 센서 (CoAP)

### 2.3 QUIC (UDP 위 TCP+TLS) 가 답
- HTTP/3
- 모바일 환경 (네트워크 전환)
- 저지연 + 신뢰성 동시

---

## 3. 메시지 경계의 차이

### TCP — 스트림

```python
# 송신
sock.send(b"hello")
sock.send(b"world")

# 수신
data = sock.recv(1024)
# data 가 "helloworld" 일 수도, "hellowo" + "rld" 일 수도
```

응용이 메시지 경계 직접 관리 (length prefix 또는 delimiter).

### UDP — 데이터그램

```python
# 송신
sock.sendto(b"hello", addr)
sock.sendto(b"world", addr)

# 수신
data, _ = sock.recvfrom(1024)   # "hello"
data, _ = sock.recvfrom(1024)   # "world"
# 또는 손실되면 한쪽이 없음
```

각 send 가 한 datagram. 부분 수신 X.

---

## 4. 헤더 비교

```
TCP (20 byte 기본):
+----------+----------+
| SrcPort  | DstPort  |
| Seq#                |
| Ack#                |
| HL Flag  | Window   |
| Checksum | Urgent   |
+----------+----------+

UDP (8 byte):
+----------+----------+
| SrcPort  | DstPort  |
| Length   | Checksum |
+----------+----------+
```

UDP 는 12 byte 더 적음 — 작은 페이로드에 큰 차이.

---

## 5. Throughput 비교 (실험적)

조건: 1 Gbps, 1 ms RTT, 1500 byte MTU

### TCP
- 첫 1 RTT — handshake
- Slow Start → Congestion Avoidance
- 안정 시 ~ 900 Mbps (overhead 빼고)

### UDP
- 즉시 line rate (1 Gbps)
- **그러나 수신 측이 처리 못 하면 손실**
- 혼잡 제어 없어서 폭주 가능

---

## 6. Latency 비교

| 시나리오 | TCP | UDP |
| --- | --- | --- |
| 첫 패킷 | 1 RTT (handshake) | 즉시 |
| 손실 + 재전송 | RTO 대기 | 응용 결정 |
| HoL blocking | 있음 (모든 stream stuck) | 없음 |
| Jitter | 작음 (재전송 시 큼) | 환경 의존 |

게임 / VoIP 는 한 패킷 손실 < HoL → UDP.

---

## 7. 멀티캐스트 / 브로드캐스트

- TCP — 1:1 만 (그룹 통신 X)
- UDP — 1:N (Multicast), 1:All (Broadcast)

→ DHCP (broadcast), IGMP, mDNS 는 UDP 필수.

---

## 8. NAT 친화도

- TCP — handshake 로 NAT 가 흐름 추적
- UDP — stateless → NAT 가 timeout 빠름

→ UDP 환경에선 keepalive 필수 (모바일 / 셀룰러).

---

## 9. 보안

| 측면 | TCP | UDP |
| --- | --- | --- |
| 암호화 | TLS (표준) | DTLS, QUIC |
| Sequence Prediction | 강한 ISN 으로 방어 | 더 어려움 (없음) |
| Spoofing | 어려움 (3-way) | 쉬움 |
| DDoS amplification | 미세 | 큼 (NTP, DNS) |
| Connection tracking | 가능 | 어려움 |

→ UDP 서비스는 BCP 38 (egress filter) 권장.

---

## 10. 응용 선택 가이드

질문해야 할 것:

```
1. 매 byte 도착이 중요? → TCP
   - 파일, JSON, HTML, DB
   
2. 늦은 데이터 = 무용? → UDP
   - 음성, 비디오, 게임 상태
   
3. 한 패킷? → UDP
   - DNS query, NTP

4. Multicast / Broadcast? → UDP
   - LAN 발견, IGMP

5. 응용 신뢰성 구현 OK? → UDP + 응용
   - QUIC, SRT, RUDP

6. 모바일 환경 / 네트워크 전환? → QUIC
   - HTTP/3
```

---

## 11. 함정

### 함정 1 — TCP 가 무조건 안전 가정
HoL Blocking 으로 실시간성 깨짐.

### 함정 2 — UDP 가 빠름만 보고 채택
응용 신뢰성 구현 비용 ≫ TCP 사용.

### 함정 3 — UDP 의 NAT keepalive 누락
30-60 초 후 흐름 끊김.

### 함정 4 — 큰 UDP 메시지
> 1472 byte 단편화. DNS DNSSEC 문제.

### 함정 5 — TCP / UDP 의 보안 동등 가정
DTLS 는 TLS 보다 덜 쓰이고 미들박스 호환 약함.

---

## 12. 학습 자료

- RFC 793, RFC 768
- **TCP/IP Illustrated Vol.1** Ch. 11, 17
- "UDP vs TCP" — Google Developers

---

## 13. 관련

- [[udp]] — UDP hub
- [[../tcp/tcp]] — TCP hub
- [[../quic/quic]] — UDP 의 모던 활용
