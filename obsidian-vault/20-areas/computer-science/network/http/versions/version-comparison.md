---
title: "HTTP 버전 비교 & Negotiation"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:30:00+09:00
tags:
  - network
  - http
  - version
  - alpn
---

# HTTP 버전 비교 & Negotiation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 비교 표 + ALPN + Alt-Svc + HTTPS DNS |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한눈에 — 전체 비교

| 측면 | 0.9 | 1.0 | 1.1 | 2 | 3 |
| --- | --- | --- | --- | --- | --- |
| **출시** | 1991 | 1996 | 1997-2022 | 2015 | 2022 |
| **Transport** | TCP | TCP | TCP | TCP+TLS | QUIC (UDP) |
| **Connection** | 1 요청/1 TCP | 1 요청/1 TCP | Keep-Alive | 1 TCP 멀티 stream | 1 QUIC 멀티 stream |
| **메시지 형식** | 텍스트 | 텍스트 | 텍스트 | 바이너리 | 바이너리 (QUIC) |
| **헤더 압축** | — | — | — | HPACK | QPACK |
| **메서드** | GET 만 | 3 (GET/POST/HEAD) | 8 (+PUT/DELETE/OPTIONS/...) | 동일 | 동일 |
| **상태 코드** | 없음 | 5xx 도입 | 모든 표준 | 동일 | 동일 |
| **Host 헤더** | — | 옵션 | **필수** | 필수 (`:authority`) | 필수 |
| **Chunked** | — | — | ✅ | DATA frame | DATA frame |
| **HoL Blocking** | — | — | 있음 | 응용 X, TCP O | **없음** |
| **0-RTT** | — | — | — | ❌ (TFO 별도) | ✅ |
| **Migration** | — | — | — | ❌ | ✅ |
| **Server Push** | — | — | — | ✅ (사장) | ✅ |
| **암호화** | — | 옵션 | 옵션 | 사실상 필수 (ALPN) | **필수** |
| **RFC** | (없음) | 1945 | **9110-9112** | **9113** | **9114** |

---

## 2. Negotiation — 어느 버전 사용?

### 2.1 ALPN (TLS handshake 동안)

```
TLS ClientHello:
  ALPN extension: ["h3", "h2", "http/1.1"]   ← 클라가 선호 순서

ServerHello:
  ALPN extension: "h2"   ← 서버가 선택
```

→ TLS 핸드셰이크 동시 협상. 한 포트 (443) 에서 여러 버전.

### 2.2 ALPN 값 표

| ALPN | 프로토콜 |
| --- | --- |
| `http/1.0` | HTTP/1.0 |
| `http/1.1` | HTTP/1.1 |
| `h2` | HTTP/2 |
| `h2c` | HTTP/2 cleartext (드묾) |
| `h3` | HTTP/3 |
| `h3-29`, `h3-Q050` | 옛 draft 버전들 |

### 2.3 Upgrade (TLS 없을 때)

```http
GET / HTTP/1.1
Host: example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64 settings>

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c
```

→ HTTP/2 over TCP (h2c). 미들박스 호환성 X — 거의 사용 안 함.

### 2.4 Alt-Svc 헤더 (RFC 7838)

```http
HTTP/2 응답:
  Alt-Svc: h3=":443"; ma=86400
```

- 클라가 캐싱 (max-age 86400 = 24h)
- 다음부터 HTTP/3 시도
- HTTP/3 발견의 표준 메커니즘

### 2.5 HTTPS DNS Record (RFC 9460, 2023)

```
$ dig example.com HTTPS

example.com.  HTTPS 1 . alpn="h3,h2" port=443
```

→ DNS 단계에서 HTTP/3 발견 — Alt-Svc 의 단점 (첫 방문 1.1/2 필수) 해결.

Apple / Cloudflare 채택 중.

---

## 3. 버전 별 사용 결정 트리

```
HTTPS + 모던 클라/서버
├ HTTP/3 사용 가능? (UDP 443 OK + Alt-Svc / HTTPS DNS)
│   ├ Yes → HTTP/3
│   └ No  → HTTP/2
└ HTTPS 안 함 / 옛 클라
    └ HTTP/1.1
```

---

## 4. 같은 의미, 다른 wire format

RFC 9110 (Semantics, 2022) 가 모든 버전 통합:
- 메서드 / 상태 / 헤더 의미 **동일**
- 캐싱 (RFC 9111) **동일**
- wire format 만 9112 (1.1) / 9113 (2) / 9114 (3)

→ 응용 코드는 버전 무관 (라이브러리가 처리).

---

## 5. 성능 비교 (실측 추세)

### 5.1 첫 방문 (DNS / TLS 처음)

| 시나리오 | 1.1 | 2 | 3 |
| --- | --- | --- | --- |
| 시간 | 기준 | -20% | -25% |

### 5.2 재방문

| 시나리오 | 1.1 | 2 | 3 |
| --- | --- | --- | --- |
| 시간 | 기준 | -30% | -50% (0-RTT) |

### 5.3 손실 환경 (Wi-Fi 약함)

| 시나리오 | 1.1 | 2 | 3 |
| --- | --- | --- | --- |
| 시간 | 기준 | -10% (TCP HoL) | -40% |

### 5.4 동시 다수 리소스

| 시나리오 | 1.1 | 2 | 3 |
| --- | --- | --- | --- |
| 시간 | 기준 (6 연결) | -40% (멀티) | -45% |

---

## 6. 호환성 / Fallback

```
HTTP/3 시도 실패 (UDP 차단 / 미들박스) → HTTP/2
HTTP/2 시도 실패 (옛 서버 / 미들박스) → HTTP/1.1
```

브라우저가 자동.

---

## 7. 응용 코드의 추상화

### Python httpx
```python
import httpx
client = httpx.Client(http2=True)        # HTTP/2
# httpx 3 는 HTTP/3 (실험적)
r = client.get("https://example.com/")
```

### curl
```bash
curl --http3 https://...
curl --http2 https://...
curl --http1.1 https://...
```

### Go
```go
// HTTP/2 자동 (TLS + ALPN)
// HTTP/3 — quic-go 별도
```

---

## 8. 보안 관점

| 버전 | 강제 TLS | 알려진 취약점 |
| --- | --- | --- |
| 1.0/1.1 | X | Response splitting, Request smuggling |
| 2 | (사실상) | HPACK HEIST, Rapid Reset (CVE-2023-44487) |
| 3 | ✅ | Migration spoofing, 0-RTT replay |

### Rapid Reset (2023)
- HTTP/2 의 stream 빠른 reset 으로 DDoS
- Cloudflare / Google / AWS 의 큰 사고
- 패치 + rate limit

---

## 9. 함정

### 함정 1 — Alt-Svc 의존
첫 방문 HTTP/2 — HTTPS DNS Record 권장.

### 함정 2 — h2c (cleartext) 시도
미들박스 호환 X. HTTPS 만.

### 함정 3 — HTTP/3 의 UDP 차단
기업 / 모바일에서 차단. Fallback 자동이지만 첫 시도 시간 손해.

### 함정 4 — 옛 클라이언트 가정
Java 8, .NET Framework 4 등은 HTTP/2 어려움.

### 함정 5 — 버전별 한계 가정
1.1 의 6 connection / 2 의 100 stream / 3 의 0-RTT replay — 각각 다름.

### 함정 6 — Negotiation 결과 측정 안 함
실제 어느 버전 쓰이는지 모니터링 — Web Almanac 같은 측정.

---

## 10. 학습 자료

- **RFC 9110** (Semantics) — 모든 버전 공통
- **RFC 9112 / 9113 / 9114** — wire format
- **HTTP/2 in Action** / **HTTP/3 explained**
- Web Almanac — 실측 통계
- Cloudflare blog — HTTP versions 시리즈

---

## 11. 관련

- [[http-1-1]], [[http-2]], [[http-3-quic]]
- [[../../tls-ssl/tls-ssl]] — ALPN
- [[../../dns/dns]] — HTTPS DNS Record
- [[versions]] — hub
