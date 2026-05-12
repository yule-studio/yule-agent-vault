---
title: "UDP — User Datagram Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:15:00+09:00
tags:
  - network
  - udp
  - layer-4
---

# UDP — User Datagram Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 / 응용 / 함정 / Lite UDP |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-4-transport/layer-4-transport|↑↑ L4 Transport]]**

---

## 1. 한 줄 정의

**비연결 + 비신뢰성** 의 가벼운 종단-종단 프로토콜. RFC 768 (1980, David Reed) —
TCP 와 함께 인터넷의 양대 transport.

---

## 2. UDP vs TCP

| 측면 | UDP | TCP |
| --- | --- | --- |
| 연결 | 비연결 | 연결 지향 (3-way) |
| 신뢰성 | 없음 | 있음 |
| 순서 | 없음 | 있음 |
| 흐름 제어 | 없음 | 있음 |
| 혼잡 제어 | 없음 | 있음 |
| 헤더 | **8 byte** | 20+ byte |
| 메시지 경계 | **보존** (1 send = 1 recv) | 보존 X (byte stream) |
| Multicast / Broadcast | ✅ | ❌ |
| 응용 예 | DNS, DHCP, NTP, 게임, VoIP | HTTP, SSH, FTP |

---

## 3. UDP 헤더 (8 byte)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Length            |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          data                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 필드

- **Source Port** (16 bit) — 선택 (0 가능)
- **Destination Port** (16 bit)
- **Length** (16 bit) — UDP 헤더 + 데이터 (max 65535)
- **Checksum** (16 bit) — IPv4 선택 (0 가능), IPv6 필수

---

## 4. UDP의 용도 — 언제 적합한가

### 4.1 짧은 요청-응답
- **DNS** — 1 패킷 query + 1 패킷 response
- **DHCP** — broadcast 필요
- **NTP** — 시간 동기

→ TCP 의 3-way handshake 비용 절약.

### 4.2 실시간 미디어
- **VoIP** (SIP/RTP)
- **화상 통화** (WebRTC)
- **라이브 스트리밍** (옛 RTMP, 모던 WebRTC)

→ 패킷 손실 < 지연. 늦은 데이터는 무용.

### 4.3 게임
- **FPS / 액션** — 60-120 Hz 상태 업데이트
- **MOBA**

→ TCP HoL Blocking 회피. Loss tolerant.

### 4.4 IoT / 센서
- **CoAP** (Constrained Application Protocol)
- **MQTT-SN** (Sensor Network)

→ 작은 메모리 / 저전력.

### 4.5 Multicast / Broadcast
- **mDNS** (Bonjour)
- **SSDP** (UPnP)
- **IGMP**

→ TCP 는 1:1 만.

### 4.6 QUIC / HTTP/3
- UDP 위에 TCP + TLS 의 모든 기능 + 더
- → 자세히 [[../quic/quic|↗ QUIC]]

---

## 5. UDP 가 부적합한 경우

- 파일 전송 (TCP / TLS / HTTP)
- 메일 (SMTP)
- 원격 접속 (SSH)
- 데이터베이스 (TCP)

신뢰성이 필요한데 UDP 쓰면 → 응용에서 신뢰성 구현 (= 또 다른 TCP 만들기).

---

## 6. 응용 레벨 신뢰성 (UDP 위)

응용이 직접 구현:
- **Sequence number** — 순서 / 중복 감지
- **ACK** — 확인
- **재전송** — timeout 또는 NACK
- **혼잡 제어** — TCP-friendly rate control

예:
- **QUIC** — UDP 위 신뢰성 (HTTP/3)
- **RAdmin / SRT** — 영상 전송
- **RUDP** — 게임 / 시뮬레이션

---

## 7. UDP Checksum

### 7.1 IPv4 — 선택
- 0 으로 채우면 비활성
- 대부분 활성 (1 의 보수합)
- Pseudo Header (src/dst IP, protocol, length) 포함

### 7.2 IPv6 — 필수
- 0 이면 패킷 폐기
- IPv6 에 IP 체크섬 없어서 더 중요

### 7.3 NAT 의 영향
- NAT 가 IP 변경 시 checksum 재계산 필수
- 약한 NAT 는 checksum 깨뜨림

---

## 8. UDP Fragmentation

큰 UDP datagram (1500 byte 초과) → IP 단편화.

### 함정
- 한 단편 손실 = 전체 폐기
- 일부 미들박스가 단편 폐기
- DNS DNSSEC 응답 (수 KB) — 단편화 흔함

### 해결
- 큰 응답 시 **TCP fallback** (DNS 의 경우)
- **DoH / DoT** — TCP/TLS 기반
- 응용 레벨 chunking

---

## 9. UDP-Lite (RFC 3828)

### 동기
음성 / 비디오는 일부 비트 오류 OK — 전체 폐기보다 일부 손상이 나음.

### 동작
- **Length 필드를 "Coverage" 로 재정의**
- 헤더 + N byte 만 checksum 보호
- 나머지는 손상돼도 통과

### 사용
- 옛 비디오 응용
- 현재 거의 사장 (모던 codec 이 에러 정정 내장)

---

## 10. DTLS — Datagram TLS

### 정의
UDP 위의 TLS — UDP 의 비신뢰성 보존하면서 암호화.

### 사용
- **WebRTC** (DTLS-SRTP)
- **OpenVPN**
- **SIP**
- **CoAP** (DTLS Secure)
- **QUIC** (TLS 1.3 사용, DTLS X)

---

## 11. SCTP — Stream Control Transmission Protocol

대안 transport:
- TCP 의 신뢰성 + UDP 의 메시지 경계
- 여러 stream 독립 (HoL 회피)
- 사용: SS7 (전화 신호), WebRTC data channel
- 자세히 → 별도 노트 (예정)

---

## 12. 함정

### 함정 1 — "UDP 가 무조건 빠름" 가정
TCP 는 혼잡 제어로 안정적 throughput. UDP 는 packet loss 폭증 가능.

### 함정 2 — 큰 UDP 단편화
> 1472 byte (Ethernet + IP header 28) 면 fragment. 미들박스 폐기 위험.

### 함정 3 — Source Port 0
일부 방화벽 폐기 — 보통 OS 가 무작위 ephemeral.

### 함정 4 — Checksum 의 약함
16 bit 만 — 강한 무결성은 응용에서.

### 함정 5 — NAT timeout
UDP 흐름 추적 어려움 — NAT 가 30 초 정도 후 잊음. **Keepalive 필수**.

### 함정 6 — Amplification 공격 가능
DNS (1:50), NTP (1:200), Memcached (1:51000) — UDP 의 stateless 특성 악용.

방어: BCP 38 (Spoofing 차단), Anycast scrubbing.

---

## 13. 도구 — UDP 테스트

```bash
# 송신 / 수신 (netcat)
# 서버
nc -u -l 5000

# 클라이언트
nc -u 127.0.0.1 5000

# iperf3 UDP
iperf3 -s
iperf3 -c server -u -b 100M

# Python
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(b'hello', ('1.2.3.4', 5000))
"
```

---

## 14. 학습 자료

- RFC 768 (UDP, 1980)
- RFC 3828 (UDP-Lite)
- **TCP/IP Illustrated Vol.1** Ch. 11
- "UDP for Game Networking" — Glenn Fiedler

---

## 15. 관련

- [[udp-vs-tcp]] — 직접 비교
- [[udp-applications]] — DNS / NTP / DHCP / VoIP
- [[../quic/quic]] — UDP 의 모던 활용
- [[../osi-7-layer/layer-4-transport/layer-4-transport]] — L4 hub
