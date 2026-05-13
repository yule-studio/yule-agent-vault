---
title: "HTTP 버전들 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:05:00+09:00
tags:
  - network
  - http
  - version
---

# HTTP 버전들 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 0.9 → 1.0 → 1.1 → 2 → 3 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 진화 한눈

| 버전 | 출시 | RFC | Transport | 핵심 |
| --- | --- | --- | --- | --- |
| **0.9** | 1991 | (없음) | TCP | GET 만, 단순 텍스트 |
| **1.0** | 1996 | RFC 1945 | TCP (단명) | 헤더 / 상태 코드 / MIME |
| **1.1** | 1997 / 2014 / 2022 | RFC 2068→2616→7230~5→**9110-9112** | TCP (Keep-Alive) | 표준 / 가장 오래 |
| **2** | 2015 | RFC 7540 → **9113** | TCP + ALPN | 바이너리 / 멀티플렉싱 / HPACK |
| **3** | 2022 | **RFC 9114** | QUIC (UDP) | HoL 해결 / 0-RTT / Migration |

자세히:
- [[http-0-9]]
- [[http-1-0]]
- [[http-1-1]]
- [[http-2]]
- [[http-3-quic]]
- [[version-comparison]]

---

## 2. 진화의 동기

### 0.9 → 1.0
- 헤더 (Content-Type / Content-Length) 필요
- 상태 코드 필요
- 메서드 추가 (POST, HEAD)

### 1.0 → 1.1
- 매 요청 새 TCP — 비효율 → **Keep-Alive**
- Host 헤더 → 가상 호스팅
- 캐싱 / 조건부 요청
- Chunked transfer

### 1.1 → 2
- 1.1 의 head-of-line blocking
- 헤더 중복 / 큰 크기
- 비효율적 동시 요청

### 2 → 3
- TCP HoL 잔존
- TCP + TLS 분리 (2 RTT handshake)
- 모바일 네트워크 전환 시 끊김
- → QUIC 으로 해결

---

## 3. 사용 현황 (2024-2025)

| 버전 | 사용률 (대형 사이트) |
| --- | --- |
| HTTP/3 | ~30% |
| HTTP/2 | ~45% |
| HTTP/1.1 | ~25% |
| HTTP/1.0 / 0.9 | < 0.1% |

Cloudflare / Akamai / Google / Meta / Apple — HTTP/3 default. 점진 증가.

---

## 4. ALPN — Application-Layer Protocol Negotiation

```
TLS handshake 시 ALPN extension 으로 협상:
  Client: "내가 지원: h2, http/1.1"
  Server: "h2"   ← HTTP/2 사용

ALPN 값:
  "http/1.1" → HTTP/1.1
  "h2"       → HTTP/2
  "h3"       → HTTP/3 (QUIC)
```

→ 한 포트 (443) 에서 여러 버전 협상.

---

## 5. Alt-Svc — Alternative Service (RFC 7838)

```
HTTP/2 응답에:
  Alt-Svc: h3=":443"; ma=86400

→ 클라가 다음부터 HTTP/3 시도
```

브라우저가 첫 방문은 HTTP/2 로, 두 번째는 HTTP/3 로 전환.

---

## 6. 버전 별 면접 핵심

- **HTTP/1.0 → 1.1**: Keep-Alive, Host, Chunked, Pipelining
- **HTTP/1.1 → 2**: Binary frame, Multiplexing, HPACK, Server Push
- **HTTP/2 → 3**: QUIC, HoL 해결, 0-RTT, Connection Migration

---

## 7. 관련

- [[../http]] — HTTP hub
- [[../../tcp/tcp]], [[../../quic/quic]] — Transport
- [[../../tls-ssl/tls-ssl]] — ALPN
