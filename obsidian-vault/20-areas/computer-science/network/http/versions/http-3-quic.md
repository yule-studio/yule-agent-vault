---
title: "HTTP/3 — 2022 RFC 9114 (QUIC 위)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:25:00+09:00
tags:
  - network
  - http
  - http-3
  - quic
---

# HTTP/3 — 2022 RFC 9114 (QUIC 위)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | QUIC 위 / QPACK / 0-RTT / Connection Migration |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

HTTP/2 의 의미 그대로 + **transport 를 TCP → QUIC (UDP) 으로** 교체. RFC 9114 (2022).
HoL 완전 해결 + 0-RTT + Connection Migration.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2012 | Google QUIC 시작 |
| 2013 | Chrome / YouTube 실험 |
| 2016 | IETF QUIC WG |
| 2018 | IETF QUIC ≠ gQUIC (분기) |
| 2020 | "HTTP/3" 이름 확정 |
| 2021 | **RFC 9000** — QUIC v1 |
| 2022 | **RFC 9114** — HTTP/3 |

자세히 → [[../../quic/quic|↗ QUIC]]

---

## 3. HTTP/2 와 차이

### 3.1 Transport

| 측면 | HTTP/2 | HTTP/3 |
| --- | --- | --- |
| Transport | TCP | **QUIC (UDP)** |
| TLS | 별도 layer | **QUIC 통합** |
| Handshake | TCP 1 RTT + TLS 1 RTT = **2 RTT** | **1 RTT (재방문 0 RTT)** |
| HoL Blocking | TCP 레벨 | **없음** |
| Migration | ❌ | **Connection ID 로 가능** |

### 3.2 Frame Type

HTTP/2 의 Frame 이 QUIC Stream 위에 매핑:

| Type | 이름 | HTTP/2 대응 |
| --- | --- | --- |
| 0x0 | DATA | DATA |
| 0x1 | HEADERS | HEADERS |
| 0x3 | CANCEL_PUSH | RST_STREAM |
| 0x4 | SETTINGS | SETTINGS |
| 0x5 | PUSH_PROMISE | PUSH_PROMISE |
| 0x7 | GOAWAY | GOAWAY |
| 0xD | MAX_PUSH_ID | — |

→ Stream 의 multiplexing, flow control 등은 **QUIC** 이 처리. HTTP/3 frame 은
응용 메시지에 집중.

### 3.3 QPACK — Header Compression (RFC 9204)

HPACK 의 문제:
- TCP 의 순서 보장에 의존 (동적 테이블 일관성)
- HTTP/3 에선 stream 독립 → 일관성 깨질 수 있음

→ **QPACK**:
- 별도 Stream 으로 동적 테이블 동기 (Encoder/Decoder stream)
- 같은 압축률 + 순서 무관

---

## 4. QUIC Stream 활용

```
QUIC connection = 1 개
QUIC stream = 수천 개 (독립)

각 HTTP/3 request/response = 1 bidirectional stream

Control Stream (uni) = SETTINGS, GOAWAY 등
Encoder Stream / Decoder Stream (uni) = QPACK 동기
```

자세히 → [[../../quic/quic-streams]]

---

## 5. 연결 수립

### 5.1 ALPN — "h3"

```
TLS 1.3 (QUIC 안) ClientHello:
  ALPN: "h3", "h3-29", ...   (드래프트 버전도 협상)

Server: "h3"
```

### 5.2 Alt-Svc 발견

```
HTTP/2 응답:
  Alt-Svc: h3=":443"; ma=86400

→ 클라가 다음부터 HTTP/3 시도
```

### 5.3 HTTPS DNS Record (RFC 9460)

```
example.com.  HTTPS 1 . alpn="h3,h2"
```

→ DNS 단에서 HTTP/3 발견 (Cloudflare).

---

## 6. 0-RTT

### 6.1 동작

```
첫 연결 후 NEW_TOKEN 받음

두 번째 연결:
  C → S: Initial + 0-RTT (GET /...)   ← 즉시 데이터
  S → C: Response (1 RTT)
```

### 6.2 위험 — Replay Attack

- 공격자가 0-RTT 패킷 캡처 → 재전송
- → **idempotent** 요청만 (GET / HEAD)
- POST 자동 거부 (브라우저)

자세히 → [[../../quic/quic-handshake#5 0-RTT]]

---

## 7. Connection Migration

```
1. Wi-Fi 에서 HTTP/3 요청 시작
2. 모바일 데이터로 전환 (Wi-Fi 끊김)
3. QUIC PATH_CHALLENGE 로 새 경로 검증
4. 같은 Connection ID — HTTP 응답 끊김 없이
```

→ 모바일 사용자 경험 ↑.

---

## 8. 도입 현황 (2024-2025)

| 서비스 | HTTP/3 |
| --- | --- |
| Google (Search, YouTube) | ✅ |
| Cloudflare (모든 사이트) | ✅ |
| Meta (FB, IG) | ✅ |
| Netflix | ✅ |
| AWS CloudFront | ✅ |
| Apple iCloud | ✅ |
| Microsoft (Azure, M365) | ✅ |

브라우저: Chrome / Edge / Firefox / Safari 모두 default 활성.

→ 인터넷 트래픽의 **30%+** HTTP/3.

---

## 9. 성능 (실측)

### 9.1 RTT 절감
- HTTP/2: 2-3 RTT (handshake) + 요청
- HTTP/3: 1 RTT (재방문 0 RTT)

### 9.2 모바일
- 네트워크 전환 시 HTTP/3 가 끊김 없음
- 4G ↔ Wi-Fi: HTTP/2 는 새 연결, HTTP/3 는 그대로

### 9.3 손실 환경
- Wi-Fi 약함 / 모바일 약전계
- HTTP/3 가 HTTP/2 보다 10-30% 빠름 (loss 환경)

### 9.4 좋은 네트워크
- 거의 차이 없음
- HTTP/3 가 CPU 더 사용 (사용자 공간 처리)

---

## 10. 구현

### 10.1 서버
- **Cloudflare** (quiche)
- **Nginx** 1.25+ (실험적)
- **Caddy** 2.6+
- **HAProxy** 2.6+
- **LiteSpeed** — 첫 보편화
- **NGINX HTTP/3 module** (별도)

### 10.2 클라이언트
- **curl** 7.66+ (libcurl + nghttp3)
- 모든 모던 브라우저
- Go quic-go, Rust quiche, Java msquic

---

## 11. 디버깅

```bash
# curl HTTP/3 (실험)
curl --http3 -v https://www.cloudflare.com/

# Chrome
chrome://flags/#enable-quic
chrome://net-export/ → log → qvis 등으로 분석

# qlog
# QUIC standard 로그 형식
# https://qvis.quictools.info/ — visualizer

# Wireshark
# Display filter: "http3"
# TLS keylog 필요 (SSLKEYLOGFILE)
```

---

## 12. 함정

### 함정 1 — UDP 차단
일부 기업 / 모바일 망이 UDP 443 차단 → HTTP/2 fallback.

### 함정 2 — 0-RTT 의 멱등성
POST 자동 거부. 응용이 POST 도 0-RTT 시도 시 위험.

### 함정 3 — 미들박스 호환
NAT / 방화벽 / WAF 가 QUIC 모름 → 차단 또는 잘못 변환.

### 함정 4 — Connection Migration 의 NAT
일부 NAT 가 IP 변경 시 새 흐름으로 → Migration 실패.

### 함정 5 — CPU 사용량
TCP 대비 더 — HW offload 발전 중.

### 함정 6 — 디버깅 어려움
모두 암호화 — SSLKEYLOGFILE + qlog 필요.

### 함정 7 — Alt-Svc 의존
첫 방문은 HTTP/2 — HTTPS DNS Record 가 해결 중.

---

## 13. 학습 자료

- **RFC 9114** (HTTP/3), **RFC 9204** (QPACK), **RFC 9460** (HTTPS DNS Record)
- **HTTP/3 explained** — Daniel Stenberg 무료 https://http3-explained.haxx.se/
- Cloudflare blog HTTP/3 시리즈
- "Manageable QUIC" — Quiche / Cloudflare

---

## 14. 관련

- [[http-2]] — 이전
- [[../../quic/quic]] — QUIC
- [[../../quic/quic-handshake]] — 0-RTT
- [[version-comparison]] — 비교
- [[../../tls-ssl/tls-ssl]] — TLS 1.3 (QUIC 내장)
