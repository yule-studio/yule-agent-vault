---
title: "HTTP/1.1 — 1997 ~ 2022 (RFC 9110-9112)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:15:00+09:00
tags:
  - network
  - http
  - http-1-1
---

# HTTP/1.1 — 1997 ~ 2022 (RFC 9110-9112)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Keep-Alive / Host / Chunked / 캐싱 / 파이프라이닝 |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

1997 RFC 2068 → 1999 RFC 2616 → 2014 RFC 7230~7235 → **2022 RFC 9110-9112** —
25 년 인터넷의 표준. 모든 모던 응용의 기반.

---

## 2. RFC 진화

| 연도 | RFC | 변경 |
| --- | --- | --- |
| 1997 | 2068 | 첫 표준 |
| 1999 | **2616** | 보편 인용 (가장 오래) |
| 2014 | **7230-7235** | 6 RFC 로 분할, 명확화 |
| 2022 | **9110 (Semantics)** + **9111 (Caching)** + **9112 (HTTP/1.1)** | 모던 통합 |

---

## 3. 1.0 대비 핵심 추가

### 3.1 Keep-Alive — Persistent Connection

```http
GET /page HTTP/1.1
Host: example.com
Connection: keep-alive    ← 기본 동작

(같은 TCP 연결 재사용)

GET /image.jpg HTTP/1.1
Host: example.com
```

- **기본 활성** (Connection: close 명시 시만 닫음)
- TCP handshake / slow start 비용 절약
- 한 페이지 = 1 TCP (수십 요청)

자세히 → [[../performance/keep-alive]]

### 3.2 Host 헤더 — Virtual Hosting

```http
GET /page HTTP/1.1
Host: api.example.com
```

- 한 IP 에 여러 도메인 호스팅 가능
- Apache `VirtualHost`, Nginx `server_name`
- 인터넷 폭발의 핵심 (IP 절약)

### 3.3 Chunked Transfer Encoding

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

4\r\n
abcd\r\n
6\r\n
hijklm\r\n
0\r\n
\r\n
```

- Content-Length 미리 알 필요 X
- 동적 응답 (streaming)
- 자세히 → [[../streaming/chunked-transfer]]

### 3.4 Pipelining (실패한 기능)

```
같은 연결로 여러 요청 동시 보냄
응답은 순서대로 받음

Request A → Server
Request B → Server (응답 안 기다림)
Request C → Server

← Response A
← Response B
← Response C
```

→ **HoL Blocking** 으로 사장. 자세히 → [[../performance/pipelining]]

### 3.5 캐싱 — Cache-Control

```http
Cache-Control: max-age=3600, public, must-revalidate
ETag: "abc123"
Last-Modified: Mon, 13 May 2026 10:00:00 GMT
```

- max-age 가 Expires (절대 시간) 의 한계 해결
- ETag — 콘텐츠 fingerprint
- 자세히 → [[../caching/caching]]

### 3.6 새 메서드

| 메서드 | 의미 |
| --- | --- |
| **PUT** | 전체 갱신 (멱등) |
| **DELETE** | 삭제 |
| **OPTIONS** | 허용 메서드 / CORS preflight |
| **TRACE** | 진단 (보안상 차단) |
| **CONNECT** | HTTPS proxy tunnel |

(이후 PATCH 추가 — RFC 5789).

### 3.7 새 상태 코드

| 코드 | 의미 |
| --- | --- |
| 100 | Continue (Expect: 100-continue) |
| 206 | Partial Content (Range) |
| 409 | Conflict |
| 412 | Precondition Failed |
| 416 | Range Not Satisfiable |
| 417 | Expectation Failed |
| 503 | Service Unavailable |
| 504 | Gateway Timeout |

### 3.8 Range Request

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1000-2000

HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-2000/50000
Content-Length: 1001
```

자세히 → [[../performance/range-requests]]

### 3.9 Content Negotiation

```http
Accept: text/html, application/json;q=0.9
Accept-Language: ko, en;q=0.8
Accept-Encoding: gzip, br
```

자세히 → [[../headers/content-negotiation]]

---

## 4. 메시지 형식

### 요청

```http
[Method] [Request-URI] [HTTP-Version] \r\n
[Header-Name]: [Header-Value] \r\n
[Header-Name]: [Header-Value] \r\n
\r\n
[Request-Body — POST/PUT 만]
```

### 응답

```http
[HTTP-Version] [Status-Code] [Reason-Phrase] \r\n
[Header-Name]: [Header-Value] \r\n
...
\r\n
[Response-Body]
```

→ **줄 끝 `\r\n` (CRLF)** + 헤더 끝 **빈 줄**.

---

## 5. HTTP/1.1 의 본질적 한계

### 5.1 Head-of-Line Blocking

```
한 TCP 연결의 응답은 순서대로:
Request 1 (느림) → Response 1 (대기)
Request 2 (빠름) → Response 2 (1 끝나기 전까지 못 보냄)
```

→ Pipelining 의 한계 — 거의 비활성.

### 5.2 헤더 중복 / 큰 크기

```http
Cookie: session=...; user_id=...; ad_tracker=...     ← 매 요청
User-Agent: Mozilla/5.0 (..) AppleWebKit/...         ← 매 요청
Accept: text/html,application/xhtml+xml,...           ← 매 요청
```

→ 매 요청에 KB 단위 헤더 (압축 X).

### 5.3 동시성 제한

브라우저가 한 도메인에 **6 연결 한계**:
- Chrome / Firefox 의 정책
- 더 많은 자원은 별도 도메인 (domain sharding)

→ HTTP/2 의 멀티플렉싱이 해결.

### 5.4 텍스트 기반의 비효율

- 파싱 비용
- 가변 길이 헤더 / 본문
- 바이너리 데이터는 Base64 등으로 인코딩

---

## 6. RFC 9110-9112 (2022) 의 정리

기능 변경 X — **명확화 + 정리**:
- **9110** Semantics — 메서드 / 상태 / 헤더의 의미 (모든 버전 공통)
- **9111** Caching — 캐싱 규칙 (모든 버전 공통)
- **9112** HTTP/1.1 — 1.1 의 wire format
- **9113** HTTP/2
- **9114** HTTP/3

→ 모든 버전이 같은 semantics, wire format 만 다름.

---

## 7. curl / tcpdump

```bash
curl -v --http1.1 https://api.example.com/

# 출력:
# > GET / HTTP/1.1
# > Host: api.example.com
# > User-Agent: curl/7.68.0
# > Accept: */*
# >
# < HTTP/1.1 200 OK
# < Date: ...
# < Content-Type: application/json
# ...
```

```bash
# Wireshark
display filter:
  http
  http.request.method == "GET"
  http.response.code == 404
  http.host contains "example"
```

---

## 8. 함정

### 함정 1 — Pipelining 활성
모던 환경에서 거의 X. HTTP/2 사용.

### 함정 2 — Connection: close 무시
응답 후 응용이 close → 다음 요청에 새 TCP.

### 함정 3 — 헤더 중복
브라우저는 자동, 응용 코드는 명시.

### 함정 4 — Chunked + Content-Length 동시
한 응답에 둘 다면 충돌. Chunked 만.

### 함정 5 — Domain Sharding
HTTP/1.1 시대 트릭, HTTP/2 에선 오히려 비효율.

---

## 9. 학습 자료

- **RFC 9110 / 9111 / 9112** (2022 통합)
- **HTTP: The Definitive Guide** Ch. 1-7
- MDN HTTP — https://developer.mozilla.org/en-US/docs/Web/HTTP
- Curl 의 HTTP/1.1 가이드

---

## 10. 관련

- [[http-1-0]] — 이전
- [[http-2]] — 후속
- [[version-comparison]] — 비교
- [[../performance/keep-alive]]
- [[../performance/pipelining]]
- [[../streaming/chunked-transfer]]
