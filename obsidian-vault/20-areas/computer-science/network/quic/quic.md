---
title: "QUIC — Quick UDP Internet Connections"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:30:00+09:00
tags:
  - network
  - quic
  - http3
---

# QUIC — Quick UDP Internet Connections

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | QUIC hub + 핸드셰이크 + Streams + HTTP/3 |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-4-transport/layer-4-transport|↑↑ L4 Transport]]**

---

## 1. 한 줄 정의

**UDP 위에 TCP + TLS 1.3 + 스트림** 의 모든 기능 통합. Google 2012 시작,
RFC 9000 (2021) 표준화. **HTTP/3 의 transport**.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2012 | Google "Quick UDP Internet Connections" 시작 |
| 2013 | Chrome / YouTube 에서 실험 |
| 2016 | IETF QUIC 워킹 그룹 |
| 2017 | "gQUIC" 와 "iQUIC" 분기 (Google vs IETF) |
| 2021 | **RFC 9000** — IETF QUIC v1 표준 |
| 2022 | **RFC 9114** — HTTP/3 표준 |
| 2024 | 모든 주요 브라우저 / 서버 / CDN 지원 |

---

## 3. QUIC 가 해결한 4 가지 문제

### 3.1 TCP의 HoL Blocking
- TCP 는 한 stream — 한 패킷 손실 → 모든 데이터 stuck
- **QUIC**: stream 독립 — 한 stream 손실이 다른 영향 X

### 3.2 TLS / TCP 의 분리
- TCP handshake (1 RTT) + TLS handshake (1-2 RTT) = 2-3 RTT
- **QUIC**: 통합 — 1 RTT 또는 0 RTT (재방문)

### 3.3 OS 의 TCP 발전 정체
- TCP 는 커널 — 새 알고리즘 배포에 OS 업데이트
- **QUIC**: 응용 공간 — 새 버전 배포 즉시

### 3.4 모바일 네트워크 전환
- TCP 는 (Src IP, Port) 변경 시 연결 끊김
- **QUIC**: Connection ID 로 식별 — IP 바뀌어도 OK

---

## 4. QUIC vs TCP+TLS

| 측면 | TCP + TLS 1.3 | QUIC |
| --- | --- | --- |
| Transport | TCP | UDP |
| 암호화 | TLS 위 | 통합 |
| Handshake | 2 RTT (TCP 1 + TLS 1) | 1 RTT |
| 0-RTT | TFO 별도 | 표준 |
| HoL Blocking | TCP 레벨 | 없음 |
| Connection migration | ❌ | ✅ |
| OS dependency | 커널 | 응용 |
| 디버깅 | tcpdump 쉬움 | 암호화 (`SSLKEYLOGFILE`) |
| 미들박스 | 호환 좋음 | 일부 차단 |

---

## 5. QUIC 패킷 구조

### 5.1 Long Header (handshake 단계)

```
+-+-+-+-+-+-+-+-+
|1|1|T T|R R|P P|       Type (Initial/Handshake/0-RTT/Retry)
+-+-+-+-+-+-+-+-+
|         Version (32)         |
+------------------------------+
| DCID Len (8) | DCID (...)    |   Destination Connection ID
+------------------------------+
| SCID Len (8) | SCID (...)    |   Source Connection ID
+------------------------------+
| Packet Number (8-32) | Payload
+------------------------------+
```

### 5.2 Short Header (handshake 이후)

```
+-+-+-+-+-+-+-+-+
|0|1|S|R|R|K|P P|     Spin bit / Key phase
+-+-+-+-+-+-+-+-+
| DCID (...)    |     Destination Connection ID (변경 가능)
+---------------+
| Packet Number (8-32) | Payload (Frames)
+---------------+
```

### 5.3 Frame Types

UDP datagram 안에 여러 frame:

| Frame | 용도 |
| --- | --- |
| **PADDING** | Anti-amplification |
| **PING** | Keepalive |
| **ACK** | 확인 (ranges) |
| **RESET_STREAM** | stream 종료 |
| **STOP_SENDING** | 수신 중지 |
| **CRYPTO** | TLS handshake 데이터 |
| **NEW_TOKEN** | 0-RTT 토큰 |
| **STREAM** | 응용 데이터 |
| **MAX_DATA / MAX_STREAM_DATA** | 흐름 제어 |
| **PATH_CHALLENGE / PATH_RESPONSE** | Migration 확인 |
| **CONNECTION_CLOSE** | 종료 |

---

## 6. QUIC Handshake

### 6.1 1-RTT (첫 연결)

```
Client → Server: Initial (ClientHello)
Server → Client: Initial (ServerHello + Cert) + Handshake (TLS Finished)
Client → Server: Handshake (Client Finished) + 응용 데이터 (1-RTT)
[양방향 데이터 가능]
```

### 6.2 0-RTT (재방문)

```
첫 연결 후 서버가 NEW_TOKEN 발급

이후 연결:
Client → Server: Initial + 0-RTT (응용 데이터 + token)
Server → Client: Initial + Handshake + 1-RTT 응답
```

→ **0 RTT 만에 데이터 송수신**. 매우 빠름.

### 6.3 0-RTT 의 위험
- Replay attack — 멱등 (GET) 만 안전
- POST 등 금지

---

## 7. Stream Multiplexing

### 7.1 Stream

```
한 QUIC connection 안에 수천 stream
각 stream 은 독립 — 손실 / 흐름 제어 분리
HTTP/3 의 각 request/response = 1 stream
```

### 7.2 Stream ID

```
[Type (2 bit)][Stream ID]
Type:
  0 — Client-initiated bidirectional
  1 — Server-initiated bidirectional
  2 — Client-initiated unidirectional
  3 — Server-initiated unidirectional
```

### 7.3 HoL Blocking 해결

```
TCP:                          QUIC:
패킷 1 (req1)                 패킷 1 (stream1 + stream2)
패킷 2 (req2) ← 손실           패킷 2 (stream1) ← 손실
패킷 3 (req3)                 패킷 3 (stream2)
패킷 4 (req4)
↓                             ↓
패킷 2 손실 시 모두 stuck     stream 1 만 stuck, stream 2 통과
```

---

## 8. Connection Migration

### 8.1 동작

```
1. 클라가 Wi-Fi (192.168.1.5) 로 연결
2. Cellular (10.0.0.5) 로 전환
3. 클라가 QUIC PATH_CHALLENGE 송신 (새 IP)
4. 서버가 응답 → 새 경로 검증
5. Connection 유지 — 같은 Connection ID
```

→ TCP 는 끊겼다 재연결, QUIC 는 매끄러움.

### 8.2 모바일에서 큰 장점
- 영상 통화 / 게임 / 다운로드 끊김 없음

---

## 9. QUIC 의 혼잡 제어

- TCP 와 같은 알고리즘 (CUBIC, BBR, NewReno)
- 응용 공간 구현
- 라이브러리 별 선택 가능 (msquic, quiche, ngtcp2, picoquic)

---

## 10. QUIC 의 암호화

- **TLS 1.3 만** (이전 버전 X)
- AEAD (AES-GCM, ChaCha20-Poly1305)
- 헤더 일부도 암호화 — middlebox 의 deep inspection 방해
- Spin bit — 외부에서 RTT 측정 가능

---

## 11. HTTP/3 = HTTP over QUIC

자세히 → [[../http/http3-quic]]

### 11.1 HTTP/2 와의 차이

| 측면 | HTTP/2 | HTTP/3 |
| --- | --- | --- |
| Transport | TCP + TLS | QUIC (UDP+TLS) |
| HoL Blocking | TCP 레벨 | 없음 |
| 0-RTT | ❌ | ✅ |
| Migration | ❌ | ✅ |
| Server Push | ✅ (제거 중) | ✅ |
| 헤더 압축 | HPACK | **QPACK** |

### 11.2 Alt-Svc (Alternative Service)

```
HTTP/2 응답 헤더:
  Alt-Svc: h3=":443"; ma=86400

→ 클라가 다음부터 HTTP/3 시도
```

---

## 12. 도입 현황 (2025)

| 서비스 | HTTP/3 |
| --- | --- |
| Google (Search, YouTube) | ✅ |
| Cloudflare | ✅ |
| Meta (Facebook, Instagram) | ✅ |
| Netflix | ✅ |
| AWS CloudFront | ✅ |
| Akamai | ✅ |
| Apple (iCloud, ...) | ✅ |
| Microsoft Azure | ✅ |
| Discord | ✅ |

브라우저:
- Chrome / Edge — 활성 (default 2020+)
- Firefox — 활성 (default 2021+)
- Safari — 활성 (default 2022+)

→ 인터넷 트래픽의 30%+ 가 QUIC (2024).

---

## 13. 함정

### 함정 1 — UDP 차단 환경
일부 기업 방화벽 / 모바일 망이 UDP 443 차단 → TCP HTTP/2 fallback.

### 함정 2 — 미들박스 호환성
NAT / Firewall 가 QUIC 모르고 폐기. 점점 좋아짐.

### 함정 3 — 0-RTT 의 replay
멱등 (idempotent) 만. POST 위험.

### 함정 4 — 디버깅 어려움
모두 암호화 → tcpdump 만으로 부족. `SSLKEYLOGFILE` + Wireshark.

### 함정 5 — Spin bit 활성
RTT 측정 부수효과 — 프라이버시 X 약간.

### 함정 6 — CPU 사용
TCP 보다 CPU 더 사용 (사용자 공간 처리). HW offload 발전 중.

---

## 14. 구현 / 라이브러리

| 구현 | 언어 | 사용 |
| --- | --- | --- |
| **quiche** (Cloudflare) | Rust | Cloudflare 프로덕션 |
| **msquic** (Microsoft) | C | Windows / Linux / Mac |
| **ngtcp2** | C | curl, nginx (실험) |
| **picoquic** | C | 학술 / 실험 |
| **quic-go** | Go | caddy |
| **aioquic** | Python | 학습 |
| **Chromium QUIC** | C++ | Chrome 내장 |

---

## 15. 도구 — QUIC 디버깅

```bash
# curl with HTTP/3 (실험적)
curl --http3 https://www.cloudflare.com/

# qlog (QUIC 표준 로그)
# - Wireshark — Display Filter: quic
# - https://qvis.quictools.info/ — visualizer

# 브라우저
chrome://flags/#enable-quic
about:networking 등으로 확인
```

---

## 16. 학습 자료

- RFC 9000 (QUIC v1), RFC 9001 (QUIC TLS), RFC 9002 (Loss/Congestion), RFC 9114 (HTTP/3)
- **HTTP/3 explained** — Daniel Stenberg (curl), https://http3-explained.haxx.se/
- Cloudflare blog QUIC 시리즈
- Google "The QUIC Transport Protocol" 논문 (2017)

---

## 17. 관련

- [[quic-handshake]] — handshake 깊이
- [[quic-streams]] — Stream 깊이
- [[../http/http3-quic]] — HTTP/3
- [[../tls-ssl/tls-ssl]] — TLS 1.3
- [[../udp/udp]] — UDP 기반
- [[../osi-7-layer/layer-4-transport/layer-4-transport]] — L4 hub
