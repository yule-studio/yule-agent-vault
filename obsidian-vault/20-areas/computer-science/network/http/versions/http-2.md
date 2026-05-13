---
title: "HTTP/2 — 2015 RFC 7540 → 9113"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:20:00+09:00
tags:
  - network
  - http
  - http-2
  - hpack
  - multiplexing
---

# HTTP/2 — 2015 RFC 7540 → 9113

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 바이너리 / Frame / Stream / HPACK / Server Push / Priority |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

Google **SPDY** (2009) 기반 IETF 표준화. **바이너리 프레이밍 + 스트림 멀티플렉싱 +
HPACK 헤더 압축** 으로 1.1 의 한계 해결. 2015 RFC 7540 → 2022 RFC 9113.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2009 | Google **SPDY** 발표 |
| 2012 | Chrome / Firefox 채택 |
| 2014 | IETF HTTP/2 워킹 그룹 |
| 2015.05 | **RFC 7540** — HTTP/2, **RFC 7541** — HPACK |
| 2022 | **RFC 9113** — 명확화 |

---

## 3. 4 가지 핵심 변화

### 3.1 Binary Framing

```
HTTP/1.1: 텍스트
"GET / HTTP/1.1\r\nHost: ...\r\n\r\n"

HTTP/2: Binary frame
+--------+--------+--------+--------+
| Length          | Type   | Flags  |
| (24)            | (8)    | (8)    |
+--------+--------+--------+--------+
| R | Stream Identifier (31)        |
+--------+--------+--------+--------+
| Frame Payload                      |
+--------+--------+--------+--------+
```

→ 파싱 빠름, 압축 가능, 오류 적음.

### 3.2 Stream Multiplexing

```
한 TCP 연결 안 수십 ~ 수천 stream
각 stream = 한 request/response
독립 — 한 stream 의 지연이 다른 영향 X (응용 레벨에서)
```

→ HTTP/1.1 의 6 connection 한계 / Pipelining 의 HoL 해결.

```
TCP 연결 1 개
├─ Stream 1: GET /index.html
├─ Stream 3: GET /style.css        (홀수 = 클라 시작)
├─ Stream 5: GET /script.js
├─ Stream 7: GET /image1.png
├─ Stream 9: GET /image2.png
└─ ...
```

### 3.3 HPACK — Header Compression (RFC 7541)

1.1 의 헤더 중복 해결:

```
방법 1 — Static Table (61 entries):
  :method=GET → index 2
  :path=/     → index 4
  user-agent  → index 58

방법 2 — Dynamic Table:
  요청마다 새 헤더 추가
  같은 헤더 재참조 — 1 byte

방법 3 — Huffman Coding:
  자주 쓰는 문자 짧은 비트
```

→ 헤더 80-90% 압축.

### 3.4 Server Push (deprecated)

```
클라가 / 요청 → 서버가:
- / (HTML)
- /style.css (예측 push)
- /script.js (예측 push)
```

이론적으로 RTT 절약. **현실**: 캐시 hit/miss 예측 어려움, 브라우저 폐기 — Chrome 2022 제거.

→ **103 Early Hints** 가 대체 (RFC 8297).

---

## 4. Frame 종류

| Type | 이름 | 용도 |
| --- | --- | --- |
| 0x0 | DATA | 페이로드 |
| 0x1 | **HEADERS** | 요청/응답 헤더 + 우선순위 |
| 0x2 | PRIORITY | Stream 우선순위 |
| 0x3 | RST_STREAM | Stream 종료 |
| 0x4 | SETTINGS | 연결 파라미터 |
| 0x5 | PUSH_PROMISE | Server Push |
| 0x6 | PING | Keepalive / RTT |
| 0x7 | GOAWAY | 연결 종료 |
| 0x8 | WINDOW_UPDATE | Flow control |
| 0x9 | CONTINUATION | 큰 헤더 분할 |

---

## 5. Stream State Machine

```
        ┌──────────┐
        │   idle   │
        └────┬─────┘
   PUSH_PROMISE / HEADERS
             ↓
        ┌──────────┐
        │   open   │
        └────┬─────┘
   ES (End Stream) / RST
             ↓
   ┌──────────┐  ┌──────────┐
   │ half-closed (remote)  │ ↔ │ half-closed (local) │
   └──────────┘  └──────────┘
             ↓
        ┌──────────┐
        │  closed  │
        └──────────┘
```

---

## 6. Flow Control

### 6.1 두 레벨
- **Connection** — 전체 흐름
- **Stream** — 각 stream 별

### 6.2 WINDOW_UPDATE
- 받는 측이 "X byte 더 받을 수 있어" 광고
- TCP 의 Window 와 같지만 응용 레벨

### 6.3 기본 / 튜닝
- 기본 65535 byte (작음 — 큰 BDP 환경 비효율)
- 서버는 보통 1 MB+ 로 늘림

---

## 7. Priority (deprecated → RFC 9218)

### 7.1 옛 (RFC 7540)
- Tree 기반 — Parent stream / Weight
- 복잡해 대부분 구현 부정확
- 폐기

### 7.2 새 (RFC 9218)
- HTTP `Priority` 헤더
- `urgency=0-7` + `incremental=true/false`
- HTTP/3 와 통합

---

## 8. 연결 시작 (Connection Preface)

### 8.1 HTTPS + ALPN (보편)

```
TLS Handshake (ALPN: "h2")
→ Client 가 PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n + SETTINGS frame
→ Server 도 SETTINGS frame
```

### 8.2 HTTP/2 over TCP (h2c, 드묾)

- Upgrade: h2c
- 거의 사용 X — HTTPS 가 표준

---

## 9. 1.1 대비 효과

### 9.1 성능

| 측면 | 1.1 | 2 |
| --- | --- | --- |
| 동시 요청 | 6 (domain 별 TCP) | 100+ stream / 1 TCP |
| 헤더 크기 | 큼 (반복) | HPACK 압축 |
| HoL Blocking | 있음 (1.1 의 큐) | 응용 레벨 해결 (TCP 레벨은 잔존) |
| 첫 페이지 로드 | — | 보통 10-30% 빠름 |

### 9.2 한계 — TCP HoL Blocking

```
TCP 의 한 패킷 손실 →
모든 stream stuck (TCP 가 순서 보장)
```

→ HTTP/3 (QUIC) 이 해결.

---

## 10. 안티패턴 — 1.1 의 트릭이 2 에선 비효율

### 10.1 Domain Sharding
- 1.1: 여러 도메인으로 6 limit 우회
- 2: 한 도메인 / 한 TCP 가 더 효율 → 여러 도메인은 TCP / HPACK 손해

### 10.2 CSS Sprites / JS Bundling
- 1.1: 요청 수 줄이기
- 2: 멀티플렉싱이라 작은 파일 다수 OK
- 단 — 변경 영향 (캐시 무효화) 고려

### 10.3 Inline Asset
- 1.1: 추가 RTT 회피
- 2: 멀티플렉싱이라 별도 파일 OK + 캐시 가능

---

## 11. 구현

### 11.1 서버
- **Nginx** 1.9.5+, **Apache** 2.4.17+
- **HAProxy** 1.8+
- **Caddy** — 기본 HTTP/2
- **Envoy / Istio** — 데이터센터

### 11.2 클라이언트
- 모든 모던 브라우저
- curl 7.43+
- Go `net/http`, Java HttpClient, Python httpx

---

## 12. 디버깅

```bash
# curl
curl --http2 -v https://example.com/

# Chrome chrome://net-export/
# Firefox about:networking#http

# nghttp (HTTP/2 도구)
nghttp -nv https://example.com/

# Wireshark: filter "http2"
# TLS keylog 필요 (암호화)
```

---

## 13. 함정

### 함정 1 — TCP HoL 잔존
2 의 멀티플렉싱은 응용만 — TCP 패킷 손실 시 여전히 stuck. HTTP/3.

### 함정 2 — Server Push 사용
대부분 효과 X / 역효과. 103 Early Hints.

### 함정 3 — 너무 많은 stream
SETTINGS_MAX_CONCURRENT_STREAMS 한계 (기본 100). 초과 시 RST_STREAM.

### 함정 4 — HPACK 의 동적 테이블 보안
**HEIST attack** — 사이즈 변화로 정보 누출. SameSite=Strict.

### 함정 5 — Priority 무시
기본 구현이 정확 X — 중요 자원이 늦게.

### 함정 6 — h2c 의 호환성
미들박스 대부분 미지원. HTTPS + ALPN.

### 함정 7 — Connection Coalescing
같은 IP + 인증서 → 한 TCP 로 합침. 의도 X 동작.

---

## 14. 학습 자료

- **RFC 9113** (HTTP/2), **RFC 7541** (HPACK), **RFC 9218** (Priority)
- **HTTP/2 in Action** (Pollard)
- **High Performance Browser Networking** Ch. 12 — 무료
- nghttp2 라이브러리 문서
- Google SPDY whitepaper

---

## 15. 관련

- [[http-1-1]] — 이전
- [[http-3-quic]] — 후속
- [[version-comparison]] — 비교
- [[../../tls-ssl/tls-ssl]] — ALPN
- [[../streaming/http2-push]]
