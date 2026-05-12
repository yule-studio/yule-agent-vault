---
title: "네트워크 (Network)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T10:00:00+09:00
tags:
  - network
  - tcp
  - http
  - osi
  - dns
---

# 네트워크 (Network)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**컴퓨터 간에 데이터를 교환** 하는 시스템. 물리적 케이블 / 무선 신호부터
TCP/IP / HTTP / TLS 까지의 계층화된 프로토콜 스택.

---

## 2. 역사 & 배경

| 연도 | 사건 |
| --- | --- |
| 1969 | ARPANET — UCLA-Stanford 첫 패킷 전송 |
| 1973 | Vinton Cerf / Bob Kahn — TCP 설계 |
| 1981 | RFC 791 — IPv4 |
| 1983 | TCP/IP — ARPANET 의 공식 프로토콜로 |
| 1989 | Tim Berners-Lee — WWW 제안 |
| 1991 | HTTP 0.9 |
| 1995 | SSL 2.0 (Netscape) |
| 1998 | IPv6 — RFC 2460 |
| 2008 | HTTP/1.1 RFC 2616 → 7230 |
| 2015 | HTTP/2 (Google SPDY 기반, RFC 7540) |
| 2018 | TLS 1.3 — RFC 8446 |
| 2022 | HTTP/3 / QUIC — RFC 9114 |

핵심 통찰: **계층화 (Layering)** — 각 계층은 위 계층의 구현을 모름. 1970 년대
ARPANET 시절부터 정착.

---

## 3. OSI 7 계층 vs TCP/IP 4 계층

| OSI 7 | TCP/IP 4 | 단위 | 대표 프로토콜 |
| --- | --- | --- | --- |
| L7 Application | Application | 메시지 | HTTP, FTP, SMTP, DNS, SSH |
| L6 Presentation | (Application) | — | TLS, JPEG, ASCII |
| L5 Session | (Application) | — | RPC, NetBIOS |
| L4 Transport | Transport | 세그먼트 | **TCP**, **UDP**, QUIC |
| L3 Network | Internet | 패킷 | **IP**, ICMP, ARP |
| L2 Data Link | Network Access | 프레임 | Ethernet, Wi-Fi, PPP |
| L1 Physical | Network Access | 비트 | Cable, Radio, Fiber |

**OSI** 는 학문적 모델, **TCP/IP** 는 실제 구현 표준. 실무는 둘을 섞어 씀.

---

## 4. 전송 계층 (L4) — TCP vs UDP

### 4.1 TCP (Transmission Control Protocol)

- **연결 지향 (Connection-oriented)** — 3-way handshake 후 통신.
- **신뢰성** — 패킷 손실 시 재전송, 순서 보장.
- **흐름 제어 (Flow Control)** — 슬라이딩 윈도우.
- **혼잡 제어 (Congestion Control)** — Slow Start, AIMD, CUBIC, BBR.

#### 3-Way Handshake
```
Client → SYN (seq=x)       → Server
Client ← SYN-ACK (seq=y, ack=x+1) ← Server
Client → ACK (ack=y+1)     → Server
[연결 수립]
```

#### 4-Way Termination
```
Client → FIN                → Server
Client ← ACK                ← Server
Client ← FIN                ← Server
Client → ACK                → Server
[TIME_WAIT 후 종료]
```

#### TIME_WAIT
- 2 × MSL (Maximum Segment Lifetime, 보통 60-120 초)
- 늦게 도착한 패킷이 다음 연결에 섞이는 것 방지
- 서버 부하 시 SO_REUSEADDR 로 해결

### 4.2 UDP (User Datagram Protocol)

- **비연결 (Connectionless)**, 신뢰성 없음
- 헤더 8 바이트 (TCP 는 20+)
- **DNS, DHCP, NTP, 게임, VoIP, 비디오 스트리밍**
- 응답성 > 신뢰성인 경우

### 4.3 비교

| 기준 | TCP | UDP |
| --- | --- | --- |
| 연결 | 연결 지향 | 비연결 |
| 신뢰성 | 보장 | 없음 |
| 순서 | 보장 | 없음 |
| 헤더 | 20+ 바이트 | 8 바이트 |
| 속도 | 느림 | 빠름 |
| 용도 | HTTP, SSH, 파일 | DNS, 게임, 미디어 |

### 4.4 QUIC (HTTP/3 의 기반)

- UDP 위에서 TCP 의 신뢰성 + TLS 1.3 통합
- 0-RTT / 1-RTT handshake
- Head-of-Line Blocking 해결 (스트림 독립)
- Google 2012 시작, RFC 9000 (2021)

---

## 5. 네트워크 계층 (L3) — IP

### 5.1 IPv4

- 32 비트 주소 (약 43 억 개) — 2011 년 IANA 소진
- 점 표기 `192.168.1.1`
- 클래스 (A/B/C) → CIDR (`/24` 표기)
- NAT 으로 사설 IP 공유

### 5.2 IPv6

- 128 비트 주소 (3.4 × 10³⁸ 개)
- 콜론 표기 `2001:0db8:85a3::8a2e:0370:7334`
- IPsec 내장, 헤더 단순화
- 보급률 2026 년 약 40-50%

### 5.3 서브넷 / CIDR

```
192.168.1.0/24
- 네트워크 주소: 192.168.1.0
- 브로드캐스트: 192.168.1.255
- 호스트 수: 254 (네트워크/브로드캐스트 제외)
- 마스크: 255.255.255.0

10.0.0.0/8 → 사설 (RFC 1918)
172.16.0.0/12 → 사설
192.168.0.0/16 → 사설
127.0.0.0/8 → loopback
```

### 5.4 라우팅

- **정적 라우팅** — 수동 설정
- **동적 라우팅** — RIP, OSPF, BGP
- **BGP (Border Gateway Protocol)** — 인터넷의 라우팅 글루
- **Longest Prefix Matching** — Radix Tree

### 5.5 ARP / ICMP

- **ARP** (Address Resolution Protocol) — IP ↔ MAC 변환
- **ICMP** — ping, traceroute, "destination unreachable"

---

## 6. 응용 계층 (L7) — HTTP

### 6.1 HTTP/1.1

- 텍스트 기반, 요청-응답
- Keep-Alive (지속 연결)
- 파이프라이닝 (잘 안 씀, Head-of-Line Blocking)
- 헤더 압축 없음 → 오버헤드

### 6.2 HTTP/2 (2015)

- **바이너리 프레이밍**
- **멀티플렉싱** — 한 TCP 연결로 여러 스트림
- **HPACK** 헤더 압축
- **Server Push** (실제로는 잘 안 씀, Chrome 2022 제거)
- 단점: TCP 레벨 Head-of-Line Blocking 여전

### 6.3 HTTP/3 (2022)

- QUIC (UDP 기반) 위
- 0-RTT 연결 재개
- 진정한 Head-of-Line 해결
- 모바일 / 네트워크 전환에 강함

### 6.4 메서드

| 메서드 | 의미 | 멱등성 | Body |
| --- | --- | --- | --- |
| GET | 조회 | ✅ | ❌ |
| POST | 생성 | ❌ | ✅ |
| PUT | 전체 갱신 | ✅ | ✅ |
| PATCH | 부분 갱신 | ❌ | ✅ |
| DELETE | 삭제 | ✅ | (선택) |
| HEAD | 헤더만 | ✅ | ❌ |
| OPTIONS | 허용 메서드 조회 / CORS | ✅ | ❌ |

### 6.5 상태 코드

| 범위 | 의미 | 대표 |
| --- | --- | --- |
| 1xx | 정보 | 100 Continue, 101 Switching |
| 2xx | 성공 | 200 OK, 201 Created, 204 No Content |
| 3xx | 리다이렉트 | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | 클라이언트 에러 | 400 Bad Request, 401 Unauth, 403 Forbidden, 404 Not Found, 429 Too Many |
| 5xx | 서버 에러 | 500 Internal, 502 Bad Gateway, 503 Unavailable, 504 Gateway Timeout |

### 6.6 캐싱

```
Cache-Control: max-age=3600, public
ETag: "abc123"
Last-Modified: Wed, 13 May 2026 09:00:00 GMT

조건부 요청:
If-None-Match: "abc123" → 304 Not Modified
If-Modified-Since: Wed, 13 May 2026 09:00:00 GMT → 304 Not Modified
```

### 6.7 쿠키 / 세션

- `Set-Cookie: sid=abc; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`
- HttpOnly — JS 접근 차단 (XSS 방어)
- Secure — HTTPS 만
- SameSite — CSRF 방어 (Strict/Lax/None)

### 6.8 CORS

```
요청 origin: https://app.com
응답 헤더:
  Access-Control-Allow-Origin: https://app.com
  Access-Control-Allow-Credentials: true
  Access-Control-Allow-Methods: GET, POST, PUT
  Access-Control-Allow-Headers: Authorization, Content-Type
```

Preflight (OPTIONS) 는 단순 요청이 아닐 때 자동 발송.

---

## 7. TLS / HTTPS

### 7.1 TLS Handshake (1.3)

```
Client → ClientHello (key share, ciphers) → Server
Client ← ServerHello + Certificate + Finished ← Server
Client → Finished → Server
[암호화 통신 시작 — 1 RTT]
```

TLS 1.2 는 2 RTT. TLS 1.3 은 0-RTT 도 가능 (재방문).

### 7.2 인증서 체인

```
Root CA (자체 서명, OS / 브라우저에 내장)
  └ Intermediate CA
      └ Server Certificate (your domain)
```

Let's Encrypt 가 무료 인증서를 보편화 (2016~).

### 7.3 HTTPS 가 보장하는 것

1. **기밀성** (Confidentiality) — 도청 방지
2. **무결성** (Integrity) — 변조 감지
3. **인증** (Authentication) — 서버가 진짜인지

---

## 8. DNS

### 8.1 계층 구조

```
. (Root)
 └ com (TLD)
   └ google.com (Authoritative)
     └ www.google.com (Subdomain)
```

### 8.2 조회 흐름

```
1. 브라우저 캐시
2. OS 캐시 (hosts 파일)
3. 라우터 / ISP DNS
4. Recursive Resolver (8.8.8.8 등)
5. Root → TLD → Authoritative
6. 응답 캐시 (TTL)
```

### 8.3 레코드 타입

| 타입 | 용도 |
| --- | --- |
| A | 도메인 → IPv4 |
| AAAA | 도메인 → IPv6 |
| CNAME | 도메인 → 도메인 (별칭) |
| MX | 메일 서버 |
| NS | 네임 서버 |
| TXT | 임의 텍스트 (SPF, DKIM) |
| SOA | Zone 정보 |
| PTR | 역방향 (IP → 도메인) |
| SRV | 서비스 (포트 포함) |

### 8.4 DNS 보안

- **DNSSEC** — 서명 검증
- **DoH** (DNS over HTTPS) — 프라이버시
- **DoT** (DNS over TLS) — 프라이버시

---

## 9. 네트워크 도구

### 9.1 진단 명령

```bash
# 연결성
ping google.com

# 경로 추적
traceroute google.com    # macOS/Linux
tracert google.com       # Windows

# 포트 / 연결 상태
netstat -an | grep LISTEN
ss -tunlp                # Linux 모던

# 패킷 캡처
tcpdump -i en0 'port 443'
wireshark                # GUI

# DNS
dig google.com
dig +trace google.com    # 전체 경로
nslookup google.com

# HTTP
curl -v https://google.com
curl -I https://google.com   # 헤더만

# TLS 검증
openssl s_client -connect google.com:443 -servername google.com
```

### 9.2 부하 / 성능 도구

- **wrk / hey** — HTTP 부하 생성
- **iperf3** — 대역폭 측정
- **mtr** — ping + traceroute 통합

---

## 10. 함정 / 안티패턴

### 함정 1 — TIME_WAIT 의 누적
짧은 연결 폭주 시 포트 소진. Connection pool 사용.

### 함정 2 — DNS TTL 무시
긴 TTL → 장애 시 빠른 fail-over 불가. 보통 60-300 초.

### 함정 3 — HTTP/2 단일 TCP의 함정
TCP 패킷 손실 시 모든 스트림 대기. HTTP/3 (QUIC) 해결.

### 함정 4 — CORS 의 credentials
`credentials: 'include'` 면 `Allow-Origin: *` 사용 불가, 명시 origin 필요.

### 함정 5 — Cookie 의 SameSite=None
2020 년부터 Chrome 이 기본 Lax. 크로스 사이트 쿠키는 명시 + Secure 필요.

### 함정 6 — TLS 인증서 갱신 누락
Let's Encrypt 는 90 일. 자동 갱신 cron 필수.

### 함정 7 — IPv4 NAT 가정
서버에서 보이는 IP 가 진짜 클라이언트 X. `X-Forwarded-For` 신뢰는 위험.

### 함정 8 — UDP 의 MTU
1500 - 헤더 ≈ 1472. 큰 페이로드는 단편화 / 손실.

---

## 11. 응용 / 토픽

### (a) Load Balancer
- L4 (TCP/UDP) — HAProxy, AWS NLB
- L7 (HTTP) — Nginx, AWS ALB
- 알고리즘: Round-robin, Least-connections, IP hash, Consistent hashing

### (b) CDN
- Edge caching → latency 50-200ms → 5-20ms
- Cloudflare, Fastly, CloudFront

### (c) 웹소켓 (WebSocket)
- HTTP Upgrade 후 full-duplex
- 실시간 chat, 게임, 알림
- `wss://` 가 TLS 기반

### (d) gRPC
- HTTP/2 + Protocol Buffers
- 양방향 스트리밍
- 마이크로서비스 RPC

### (e) MQTT
- IoT pub-sub
- 작은 페이로드, 신뢰성 QoS 0/1/2

### (f) WebRTC
- 브라우저 간 P2P 비디오/오디오
- ICE + STUN + TURN

### (g) BGP / 라우팅
- 자율 시스템 (AS) 간 라우팅
- Facebook 2021 장애 — BGP 광고 철회

---

## 12. 대표 면접 질문

1. "주소창에 google.com 입력 후 일어나는 일을 설명하라" — DNS → TCP → TLS → HTTP → 렌더링.
2. **TCP vs UDP** — 신뢰성 / 연결성 / 사용 사례.
3. **3-way handshake / 4-way termination** — 다이어그램.
4. **HTTP/1.1 vs HTTP/2 vs HTTP/3** — 멀티플렉싱 / HoL.
5. **HTTPS 가 보장하는 것** — 기밀성 / 무결성 / 인증.
6. **CORS 동작 원리** — Preflight / Origin / credentials.
7. **세션 vs 토큰** — Stateful vs Stateless.
8. **GET vs POST** — 멱등성 / 캐싱 / Body.
9. **REST 의 원칙** — Uniform Interface / Stateless / Cacheable.
10. **Load Balancer 알고리즘** — Round-robin / Consistent Hash.

---

## 13. 학습 자료

- **TCP/IP Illustrated Vol.1** (Stevens) — 고전
- **High Performance Browser Networking** (Ilya Grigorik) — 무료 https://hpbn.co
- **Computer Networking: A Top-Down Approach** (Kurose / Ross)
- **RFC 9110/9112/9113/9114** — HTTP 최신 RFC
- [Cloudflare Learning](https://www.cloudflare.com/learning/)
- [MDN Web Docs — HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)

---

## 14. 관련

- [[../security-theory/security-theory]] — TLS, 인증, OAuth
- [[../distributed-systems/distributed-systems]] — CAP, Consistency
- [[../operating-system/operating-system]] — Socket, I/O Multiplexing
- [[../computer-science|↑ computer-science]]
