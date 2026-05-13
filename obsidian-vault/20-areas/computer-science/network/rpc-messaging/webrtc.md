---
title: "WebRTC — Peer-to-Peer 통신"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:30:00+09:00
tags:
  - network
  - webrtc
  - p2p
  - real-time
---

# WebRTC — Peer-to-Peer 통신

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | STUN / TURN / ICE / SDP |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **표준** | W3C / IETF |
| **시작** | Google (2011 — GIPS 인수) |
| **사용** | 화상 / 음성 / 데이터 |
| **포트** | UDP (any), TLS, DTLS-SRTP |

---

## 1. 한 줄 정의

**브라우저 ↔ 브라우저** 직접 통신 — 화상 / 음성 / 데이터 채널. 서버 우회 → 낮은 latency, 대역 절약.

---

## 2. 구성 요소

| | 설명 |
| --- | --- |
| **getUserMedia** | 카메라 / 마이크 캡쳐 |
| **RTCPeerConnection** | P2P 연결 |
| **RTCDataChannel** | 데이터 채널 |
| **SDP** | Session Description Protocol (협상) |
| **ICE** | Interactive Connectivity Establishment |
| **STUN** | Session Traversal Utilities (외부 IP 알기) |
| **TURN** | Traversal Using Relays (NAT 우회 relay) |

---

## 3. NAT — 가장 큰 문제

### 문제
- 대부분 사용자 — 사설 IP (NAT 뒤)
- 외부에서 직접 connect X

### 해결 — ICE (Interactive Connectivity Establishment)
- 여러 candidate (IP/Port 후보) 시도
- 가장 빠른 / 성공한 path 선택

### ICE Candidate 유형
| 유형 | 설명 |
| --- | --- |
| **host** | 로컬 IP |
| **srflx (server reflexive)** | STUN 으로 알아낸 외부 IP |
| **prflx (peer reflexive)** | peer 가 본 내 IP |
| **relay** | TURN 서버 경유 |

---

## 4. STUN — Session Traversal Utilities

### 정의
- 클라이언트의 외부 IP / port 알려줌
- 매우 가벼움 (UDP)

### 흐름
```
Client → STUN: 내 외부 IP/port 가 뭐?
STUN → Client: 1.2.3.4:5000
```

### 공용 STUN
- stun.l.google.com:19302
- stun.cloudflare.com:3478

### Symmetric NAT 에서 실패
- NAT 가 dest 별 다른 port — STUN 의 port 와 P2P port 다름
- TURN fallback 필요

---

## 5. TURN — Traversal Using Relays

### 정의
- NAT 뚫기 실패 시 — TURN 서버가 relay
- 모든 트래픽 TURN 통과

### 단점
- 대역 / 비용 (서버 부담)
- Latency ↑

### 비율
- 일반 — 10-20% 의 호출만 TURN
- 엔터프라이즈 / 학교 — 더 높음

### 서버
- **coturn** (open source)
- **AWS Kinesis Video Streams**
- **Cloudflare TURN**

---

## 6. SDP — Session Description Protocol

### 정의
- 미디어 / 코덱 / 네트워크 정보 교환
- 텍스트 포맷 (RFC 4566)

### 예
```
v=0
o=alice 2890844526 2890844526 IN IP4 host.example.com
s=Session
c=IN IP4 host.example.com
t=0 0
m=audio 49170 RTP/AVP 0 96
a=rtpmap:0 PCMU/8000
a=rtpmap:96 opus/48000/2
m=video 51372 RTP/AVP 31
a=rtpmap:31 H261/90000
```

### Offer / Answer
```
Caller: createOffer() → SDP
Callee: setRemoteDescription(offer) → createAnswer() → SDP
Caller: setRemoteDescription(answer)
```

---

## 7. Signaling

### 정의
- 두 peer 의 SDP / ICE candidate 교환 channel
- WebRTC 표준 외 — 별도

### 옵션
- WebSocket (가장 흔함)
- HTTP polling
- XMPP / SIP

### 흐름
```
Caller: SDP offer → Signaling server → Callee
Callee: SDP answer → Signaling server → Caller

(둘 다) ICE candidates → 교환
        ↓
P2P 연결 확립
```

---

## 8. 핸드셰이크 — 전체

```
1. getUserMedia (카메라 / 마이크)
2. RTCPeerConnection 생성
3. addTrack (미디어 추가)
4. createOffer → SDP
5. setLocalDescription(offer)
6. Signaling 으로 → 상대
7. 상대 setRemoteDescription(offer)
8. 상대 createAnswer → SDP
9. 상대 setLocalDescription(answer)
10. Signaling → 자기
11. setRemoteDescription(answer)
12. ICE candidates 교환 (동시 진행)
13. P2P 연결 — RTP / DTLS-SRTP
14. 미디어 전송
```

---

## 9. 미디어 — DTLS-SRTP

### DTLS (RFC 6347)
- UDP 위의 TLS
- 키 교환

### SRTP (RFC 3711)
- Real-time Transport Protocol + 암호화
- 미디어 (audio / video)

### 코덱
- **Audio** — Opus (기본), G.711, AAC
- **Video** — VP8 / VP9 / H.264 / AV1

---

## 10. RTCDataChannel — P2P 데이터

```javascript
const pc = new RTCPeerConnection();
const channel = pc.createDataChannel('chat', {
    ordered: true,
    maxRetransmits: 3
});

channel.onopen = () => channel.send('Hello');
channel.onmessage = (e) => console.log(e.data);
```

### 옵션
- **ordered** — 순서 보장 (TCP-like) vs 비순서 (UDP-like)
- **maxRetransmits** — 재전송 횟수
- **maxPacketLifeTime** — 메시지 TTL

### 사용
- 게임 (위치 데이터)
- 파일 전송 (P2P)
- 채팅 (signaling 없이)

---

## 11. SFU / MCU — 다자간

### Mesh (peer 간 직접)
- N x (N-1) connection
- 클라이언트 부하 ↑
- 4-5 명 한계

### SFU (Selective Forwarding Unit)
```
A → SFU → B, C, D
B → SFU → A, C, D
```

- SFU 가 미디어 forward (변환 X)
- 더 많은 참가자
- 클라이언트 — 자기 stream 만 upload

### MCU (Multipoint Control Unit)
```
A, B, C, D → MCU → 합성 video → 각자
```

- MCU 가 mixing
- 클라이언트 — 한 stream 만 다운로드
- MCU 부담 ↑

### 제품
- **Janus** (open source)
- **mediasoup** (Node.js)
- **LiveKit**
- **Jitsi**
- **Daily.co / Twilio Video / Agora**

---

## 12. 사용 사례

- 화상 회의 (Google Meet, Zoom — 일부)
- 음성 통화 (Discord, Slack)
- 파일 전송 (P2P)
- 게임 (low latency)
- 라이브 스트리밍 (Mux, LiveKit)

---

## 13. 보안

### 강제 암호화
- DTLS-SRTP — 필수
- 평문 X

### Origin
- HTTPS 만 — getUserMedia (보안)

### IP leak
- WebRTC — proxy 우회 — real IP 노출 가능
- VPN 사용자 — disable 권장

---

## 14. WebRTC vs WebSocket

| | WebRTC | WebSocket |
| --- | --- | --- |
| **연결** | P2P (서버 우회 가능) | 클라 ↔ 서버 |
| **전송** | UDP (SCTP/DTLS-SRTP) | TCP |
| **Latency** | 매우 낮음 | 보통 |
| **미디어** | native (audio/video) | manual encode |
| **복잡** | 매우 복잡 | 단순 |
| **사용** | 화상 / 음성 / 게임 | 채팅 / notification |

---

## 15. 함정

### 함정 1 — NAT 의 복잡
Symmetric NAT — STUN fail. TURN fallback 필요. TURN 서버 운영 부담.

### 함정 2 — Signaling 비표준
구현 마다 다름. WebSocket / Firebase / 자체.

### 함정 3 — Mesh 의 한계
4-5 명 이상 — SFU / MCU.

### 함정 4 — 코덱 호환
브라우저 마다 — VP9 / AV1 지원 다름. H.264 안전.

### 함정 5 — Mobile 의 battery / 데이터
WebRTC 무거움. WiFi 권장.

### 함정 6 — 방화벽 / 회사망
UDP 차단 / 9000+ port 차단 — fail. TURN over TCP 443 (TLS).

### 함정 7 — Echo / 음향 피드백
Echo cancellation / noise suppression — getUserMedia constraint 활용.

---

## 16. 학습 자료

- WebRTC.org (Google)
- "High Performance Browser Networking" — WebRTC
- "WebRTC: APIs and RTCWEB Protocols of the HTML5 Real-Time Web"
- web.dev — WebRTC 가이드

---

## 17. 관련

- [[rpc-messaging]] — Hub
- [[websocket]] — Signaling
- [[../udp/udp]] — UDP 기반
- [[../quic/quic]] — 비슷한 UDP / TLS / multiplexed
