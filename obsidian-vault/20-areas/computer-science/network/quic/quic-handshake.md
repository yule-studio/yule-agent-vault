---
title: "QUIC Handshake — 1-RTT, 0-RTT, Connection ID"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:35:00+09:00
tags:
  - network
  - quic
  - handshake
  - tls
---

# QUIC Handshake — 1-RTT, 0-RTT, Connection ID

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Handshake 단계 / Cryptographic Handshake / 0-RTT / Connection ID |

**[[quic|↑ QUIC]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

QUIC = TCP + TLS 1.3 통합 → **1 RTT** 또는 **0 RTT** 연결 수립.

---

## 2. 1-RTT 흐름

```
Client                                  Server
  |                                       |
  |---- Initial(ClientHello) ----------→  |
  |                                       |
  |  ←--- Initial(ServerHello)          --|
  |  ←--- Handshake(Cert, Finished)     --|
  |                                       |
  |---- Handshake(Finished) ----------→   |
  |---- 1-RTT(Application data) ------→   |
  |                                       |
  |  ←--- 1-RTT(Application data) --------|
```

1 RTT 만에 양방향 데이터 가능 — TCP+TLS 의 2-3 RTT 보다 빠름.

---

## 3. 4 가지 패킷 타입

### 3.1 Initial
- 첫 핸드셰이크
- TLS ClientHello / ServerHello 운반 (CRYPTO frame)
- Token 포함 (Retry / NEW_TOKEN 후)

### 3.2 Handshake
- TLS Finished, 인증서
- 1-RTT 키 도출

### 3.3 0-RTT
- 첫 연결 후 재방문 시
- 응용 데이터 + Early Data (TLS)

### 3.4 1-RTT
- 핸드셰이크 완료 후
- 응용 데이터 + ACK

### 3.5 Retry
- 서버가 amplification 방어 시
- 클라가 token 받아 재시도

---

## 4. Cryptographic Levels

QUIC 은 키 4 단계:

| 단계 | 키 도출 | 보호 |
| --- | --- | --- |
| **Initial** | Version + DCID 기반 (공개) | Salt + AEAD |
| **0-RTT** | 이전 세션 키 | AEAD |
| **Handshake** | TLS handshake | AEAD |
| **1-RTT** | TLS Application Traffic Secret | AEAD |

→ Initial 도 암호화되지만 키가 공개 — **인증** 만 보장 (도청 X 보장 아님).

---

## 5. 0-RTT (제로 RTT 데이터)

### 5.1 동작

```
첫 연결 (Token 발급):
  C ↔ S: 1-RTT handshake
  S → C: NEW_TOKEN frame (다음번에 쓸 토큰)
  C: Token + Session Resumption 정보 저장

재방문:
  C → S: Initial (ClientHello + Token) + 0-RTT (Application data)
  S → C: Initial (ServerHello) + Handshake (Finished) + 1-RTT (응답)
  C → S: Handshake (Finished) + 1-RTT (추가 데이터)
```

→ **첫 패킷에 응용 데이터 포함** — 즉시 시작.

### 5.2 Replay Attack 위험

```
공격자가 0-RTT 패킷 캡처 후 재전송 → 서버가 또 처리
```

방어:
- 응용이 **idempotent** 요청만 (GET, HEAD)
- TLS 의 Anti-Replay Buffer
- 시간 윈도우 제한

### 5.3 사용
- HTTP/3 의 GET — OK
- HTTP/3 의 POST / PUT — 금지 (브라우저가 자동)

---

## 6. Connection ID — 마이그레이션의 핵심

### 6.1 정의

```
TCP: (Src IP, Src Port, Dst IP, Dst Port) 의 4-tuple 로 식별
QUIC: 별도의 Connection ID (0-160 bit)
```

→ IP / Port 가 변해도 같은 connection.

### 6.2 Destination CID / Source CID

- **DCID** (Destination) — 받는 측의 ID (받는 사람이 자기를 부르는 이름)
- **SCID** (Source) — 보내는 측의 ID

양쪽이 자기 ID 선택 후 교환.

### 6.3 Connection Migration

```
1. Client (Wi-Fi 192.168.1.5) ↔ Server (1.2.3.4:443)
   QUIC Connection ID: 0xABCDEF

2. Client 가 Cellular (10.0.0.5) 로 전환
3. Client 가 PATH_CHALLENGE frame 송신 (새 IP에서)
4. Server 가 PATH_RESPONSE 송신
5. 경로 검증 OK → 새 IP/Port 채택 (Connection ID 같음)
```

### 6.4 NAT Rebinding

NAT 가 timeout 후 다른 포트로 매핑 변경:
- TCP: 연결 끊김
- QUIC: PATH_CHALLENGE / PATH_RESPONSE 로 자동 회복

---

## 7. Amplification Attack 방어

### 7.1 위협
QUIC 는 UDP — Spoofed source IP 로 SYN 같은 패킷 → 서버가 큰 응답 → DDoS.

### 7.2 방어 — 3× 제한

```
서버가 클라 검증 전까지 받은 byte 의 3 배까지만 송신 가능
```

### 7.3 Retry
큰 응답 필요시 서버가 Retry + Token → 클라가 token 들고 재시도 → 검증.

---

## 8. Version Negotiation

```
1. Client → Server: Initial (Version=v1)
2. Server: v1 미지원 → Version Negotiation Packet (지원 버전 목록)
3. Client: 공통 버전으로 재시도
```

QUIC v2 (RFC 9369) 가 일부 도입 — Greasing 으로 호환성 테스트.

---

## 9. Key Update

```
일정 packet 후 새 키로 갱신
Spin bit (1 bit) 토글 — Key phase
양쪽이 동시 갱신
```

오랫동안 같은 키 = 분석 가능 → 주기적 갱신.

---

## 10. tcpdump / Wireshark 로 QUIC

```bash
sudo tcpdump -i en0 -nn 'udp port 443'

# Wireshark
# Display filter: quic
# Initial packet 의 일부만 평문 — 나머지 암호화
```

복호화하려면 `SSLKEYLOGFILE`:

```bash
export SSLKEYLOGFILE=~/quic-keys.log
chrome   # 또는 firefox

# Wireshark — Preferences → Protocols → TLS → (Pre)-Master-Secret log filename
```

---

## 11. 함정

### 함정 1 — 0-RTT 의 replay 보안
서버 응용이 idempotent 검증 안 하면 위험.

### 함정 2 — Connection ID 길이 0
일부 deployment 가 길이 0 — Migration 안 됨.

### 함정 3 — Amplification 3× 무시
구현 오류로 큰 응답 → DDoS 도구화.

### 함정 4 — 미들박스 무지
일부 NAT 가 새 IP/Port 의 UDP 차단 → Migration 실패.

### 함정 5 — TLS 1.3 만
구식 클라 (Java 7, 옛 mobile) 호환 X.

---

## 12. 학습 자료

- RFC 9000 Section 7-8 (Handshake), RFC 9001 (TLS in QUIC), RFC 9002 (Recovery)
- "QUIC Implementation: Drafts and Issues" — Cloudflare
- "0-RTT and Anti-Replay" — TLS 1.3 RFC 8446

---

## 13. 관련

- [[quic]] — QUIC hub
- [[quic-streams]] — Stream 깊이
- [[../tls-ssl/tls-ssl]] — TLS 1.3
- [[../http/http3-quic]] — HTTP/3
