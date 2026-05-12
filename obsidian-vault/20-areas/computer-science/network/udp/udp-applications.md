---
title: "UDP 응용 — DNS / DHCP / NTP / VoIP / 게임"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:25:00+09:00
tags:
  - network
  - udp
  - dns
  - dhcp
  - ntp
---

# UDP 응용 — DNS / DHCP / NTP / VoIP / 게임

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | UDP 의 주요 응용 분야 |

**[[udp|↑ UDP]]** · **[[../network|↑↑ network hub]]**

---

## 1. DNS (Domain Name System, Port 53)

### 1.1 왜 UDP

- 작은 query (수십 byte) + 작은 response (수백 byte)
- TCP handshake = 3 RTT vs UDP = 1 RTT
- 모든 DNS resolution 의 첫 시도

자세히 → [[../dns/dns]]

### 1.2 TCP fallback
- 응답 > 512 byte (RFC 1035) → TC (Truncated) flag → 클라가 TCP 재시도
- EDNS0 (RFC 6891) — 4096 byte 까지 UDP
- DNSSEC 응답은 큼 → 종종 TCP

### 1.3 DoH / DoT
- DNS over HTTPS (TCP/TLS)
- DNS over TLS (TCP)
- 프라이버시 + 변조 방지

---

## 2. DHCP (Dynamic Host Configuration, Port 67/68)

### 2.1 동작 (DORA)

```
1. DISCOVER — 클라이언트 broadcast (UDP 67)
2. OFFER — DHCP 서버 응답 (UDP 68)
3. REQUEST — 클라이언트가 offer 수락
4. ACK — 서버 확정
```

### 2.2 왜 UDP / Broadcast
- 클라이언트가 자기 IP 아직 모름 → broadcast 필요
- TCP 는 IP 가 있어야 동작

### 2.3 필드
- IP 주소
- Subnet mask
- Default gateway
- DNS 서버
- Lease time

### 2.4 DHCPv6
- IPv6 는 SLAAC 가 주, DHCPv6 는 보완
- UDP 547 (server) / 546 (client)

---

## 3. NTP (Network Time Protocol, Port 123)

### 3.1 왜 UDP
- 작은 패킷 (48 byte)
- 한 RTT 응답 — TCP handshake 가 시간 오차 증가

### 3.2 동작

```
T1 — 클라 송신 시간
T2 — 서버 수신 시간
T3 — 서버 송신 시간
T4 — 클라 수신 시간

Offset = ((T2 - T1) + (T3 - T4)) / 2
Delay  = (T4 - T1) - (T3 - T2)
```

### 3.3 계층 (Stratum)
- **Stratum 0** — 원자 시계 / GPS
- **Stratum 1** — 직접 연결
- **Stratum 2** — Stratum 1 동기
- ...
- **Stratum 16** — 동기 X

### 3.4 정확도
- LAN — μs
- WAN — ms
- 모바일 — 10s ms

### 3.5 PTP (Precision Time Protocol)
- IEEE 1588 — ns 단위
- HW 타임스탬프
- 데이터센터 / HFT

---

## 4. SNMP (Port 161/162)

### 4.1 용도
- 네트워크 장비 모니터링
- 라우터 / 스위치 / 서버 상태

### 4.2 버전
- v1 / v2c — 평문 community string ❌
- **v3** — 인증 + 암호화 ✅

### 4.3 동작
- Manager → Agent: GetRequest / SetRequest (161)
- Agent → Manager: Trap (162) — 이벤트 알림

---

## 5. Syslog (Port 514)

### 5.1 동작
- UDP — fire-and-forget
- TCP / TLS — 신뢰성 (RFC 5425)

### 5.2 형식
```
<priority>timestamp hostname app[pid]: message
```

Priority = facility × 8 + severity (0-7).

---

## 6. VoIP / RTP (Real-time Transport Protocol)

### 6.1 RTP (RFC 3550)
- UDP 위 미디어 전송
- Sequence Number, Timestamp, SSRC

### 6.2 동작
```
SIP (TCP/UDP) — Signaling — 통화 수립
RTP (UDP) — 미디어 (음성/영상)
RTCP (UDP) — 품질 보고
```

### 6.3 Codec
- 음성: G.711, G.729, Opus
- 영상: H.264, H.265, VP9, AV1

### 6.4 WebRTC
- DTLS-SRTP (UDP + 암호화)
- ICE (STUN + TURN) for NAT

---

## 7. 게임 (Game Networking)

### 7.1 왜 UDP
- 매 16ms (60 Hz) 상태 업데이트
- 늦은 패킷 = 무용
- TCP HoL Blocking 회피

### 7.2 응용 신뢰성
- Sequence # (응용 정의)
- 중요 메시지만 ACK + 재전송
- 상태 동기는 가장 최근만

### 7.3 압축
- Delta encoding (이전과 차이만)
- Bit packing
- Range coding

---

## 8. CoAP — IoT 의 HTTP

### 8.1 RFC 7252

- HTTP 의 REST 모델을 UDP 로
- 작은 헤더 (4 byte)
- 작은 메시지 (수백 byte)

### 8.2 메서드

| HTTP | CoAP |
| --- | --- |
| GET | GET |
| POST | POST |
| PUT | PUT |
| DELETE | DELETE |

### 8.3 신뢰성
- Confirmable (CON) — ACK 필요
- Non-confirmable (NON) — fire-and-forget

### 8.4 DTLS
- Secure CoAP (coaps://)

---

## 9. mDNS / SSDP / Bonjour

### 9.1 mDNS (RFC 6762)
- 224.0.0.251:5353
- LAN 안 .local 도메인 해결
- AirDrop, AirPrint, HomeKit

### 9.2 SSDP (Simple Service Discovery)
- 239.255.255.250:1900
- UPnP 의 발견
- Smart TV, Sonos

---

## 10. QUIC (Port 443 UDP)

UDP 위에 TCP + TLS + 더:
- 자세히 → [[../quic/quic]]

---

## 11. TFTP — Trivial File Transfer (Port 69)

### 11.1 용도
- 부트 로더 / PXE boot
- 네트워크 장비 펌웨어
- 단순 — 인증 X

### 11.2 동작
- Block 단위 (512 byte default)
- Block 마다 ACK
- 단순 stop-and-wait

---

## 12. 학습 자료

- RFC 1035 (DNS), RFC 2131 (DHCP), RFC 5905 (NTPv4)
- RFC 3550 (RTP), RFC 7252 (CoAP), RFC 6762 (mDNS)
- **TCP/IP Illustrated Vol.1** Ch. 11+
- "WebRTC in the real world"

---

## 13. 관련

- [[udp]] — UDP hub
- [[../dns/dns]] — DNS 깊이
- [[../mail/mail]] — 메일 (TCP)
- [[../rpc-messaging/rpc-messaging]] — gRPC / MQTT
