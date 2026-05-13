---
title: "TRACE & CONNECT"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:00:00+09:00
tags:
  - network
  - http
  - trace
  - connect
  - proxy
---

# TRACE & CONNECT

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | TRACE (진단 + XST) + CONNECT (Proxy tunnel) |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

# Part A. TRACE — 진단 (보통 차단)

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ✅ |
| **멱등** | ✅ |
| **캐시 가능** | ❌ |
| **Body** | ❌ |
| **도입** | HTTP/1.1 |

## A.1 한 줄 정의

**중간 경유지를 거치며 요청이 어떻게 변경되는지 추적**. Loopback 진단. 보안상 대부분 차단.

## A.2 요청 / 응답

```http
TRACE / HTTP/1.1
Host: example.com
Max-Forwards: 5

```

```http
HTTP/1.1 200 OK
Content-Type: message/http

TRACE / HTTP/1.1
Host: example.com
Max-Forwards: 5
Via: 1.1 proxy1 (Apache/2.4), 1.1 proxy2
X-Forwarded-For: 1.2.3.4
Cookie: session=abc123
```

→ 서버가 **받은 요청 그대로** 응답 body 로 echo.

## A.3 Max-Forwards

```
Max-Forwards: 5
```

각 proxy 마다 -1. 0 되면 그 proxy 가 응답 (더 안 forward).

→ traceroute 비슷한 효과 (proxy chain 진단).

## A.4 XST — Cross-Site Tracing 공격

```
공격: 피해자 브라우저가 HttpOnly 쿠키 있음 → JS 에서 직접 X
TRACE 사용:
  XMLHttpRequest.open("TRACE", "/")
  요청에 쿠키 자동 포함 (브라우저)
  서버가 echo → 응답 body 에 쿠키 → JS 가 읽음
```

→ HttpOnly 우회. 2003 Jeremiah Grossman 공개.

## A.5 방어

```
대부분 서버 / 방화벽이 TRACE 차단:
  Apache: TraceEnable off
  Nginx: 자체 차단
  WAF / 방화벽: 405 또는 차단
```

→ 모던 브라우저도 TRACE XHR 차단 (2007 이후).

## A.6 결론

- **TRACE 거의 사용 X**
- 보안 스캐너의 점검 항목 ("Method allowed: TRACE" 발견 시 취약점)
- 진단은 `curl -v`, `tcpdump`, Wireshark 가 표준

---

# Part B. CONNECT — Proxy Tunnel

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ❌ |
| **멱등** | ❌ |
| **캐시 가능** | ❌ |
| **Body** | ❌ (request) / Tunnel data |
| **도입** | HTTP/1.1 |

## B.1 한 줄 정의

**HTTP proxy 를 통한 TCP tunnel 수립**. HTTPS over HTTP proxy 의 표준 방법.

## B.2 흐름

```
1. 클라 → Proxy: CONNECT origin-host:port HTTP/1.1
2. Proxy: origin-host:port 로 TCP 연결
3. Proxy → 클라: 200 Connection Established
4. 이후 클라 ↔ Proxy ↔ Origin 의 raw TCP tunnel
```

## B.3 예 — HTTPS over Proxy

```http
CONNECT api.example.com:443 HTTP/1.1
Host: api.example.com:443
User-Agent: curl/7.68.0
Proxy-Authorization: Basic dXNlcjpwYXNz

```

```http
HTTP/1.1 200 Connection Established

(이후 TLS handshake + HTTPS 데이터 — proxy 가 못 봄)
```

→ End-to-end 암호화 유지하면서 proxy 통과.

## B.4 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 Connection Established** | Tunnel 수립 |
| **407 Proxy Authentication Required** | 인증 필요 |
| **502 Bad Gateway** | Origin 연결 실패 |
| **504 Gateway Timeout** | Origin 응답 없음 |

## B.5 사용 사례

### B.5.1 HTTPS Proxy
- 회사 / ISP / Country firewall 의 proxy 통과
- 브라우저 / curl 자동 사용
- `HTTPS_PROXY=http://proxy:3128 curl https://...`

### B.5.2 HTTP/2 의 CONNECT
- 한 HTTP/2 stream 위 TCP tunnel
- 예: HTTP/2 WebSocket proxy

### B.5.3 HTTP/3 의 CONNECT-UDP (RFC 9298)
- QUIC 위에 UDP tunnel
- Apple Private Relay, MASQUE

### B.5.4 Tor / VPN
- HTTP proxy 와 결합

## B.6 보안 위험

### B.6.1 Open Proxy
- 인증 없는 CONNECT — 익명 통신 도구화
- 스팸 / 공격 경유지

### B.6.2 Port restriction
```
CONNECT origin:80   → 일반 HTTP (목적 모호)
CONNECT origin:443  → HTTPS (정상)
CONNECT origin:25   → SMTP (스팸 위험)
CONNECT origin:6667 → IRC
```

→ 대부분 proxy 가 443 만 허용 (`allowed_ports = 443`).

### B.6.3 CONNECT 의 wildcard
```
CONNECT *:443  ← 모든 host 통과
```

특정 도메인 화이트리스트 권장.

### B.6.4 MITM proxy (Burp, Squid SSL bump)
- 의도적 — CONNECT 후 자체 TLS 종료
- 개발 / 디버깅용. 사용자 동의 필요.

## B.7 Proxy 설정 예

```bash
# curl
curl -x http://proxy:3128 https://example.com/
curl --proxy-user user:pass -x http://proxy:3128 https://...

# 환경변수
export HTTPS_PROXY=http://proxy:3128
curl https://example.com/

# Squid 설정 (allowed_ports)
acl SSL_ports port 443
acl Safe_ports port 80
http_access deny CONNECT !SSL_ports
```

## B.8 함정

### 함정 1 — CONNECT 응답 후 헤더
일부 server 가 200 응답에 헤더 더 추가 — 클라가 raw data 와 혼동.

### 함정 2 — Proxy timeout
긴 tunnel (Slack / Discord WebSocket) — proxy 가 idle timeout. Keep-alive.

### 함정 3 — Proxy 의 CONNECT 로깅
HTTPS 내용은 모르지만 host:port + 시간 + 크기 노출.

### 함정 4 — Port 25 (SMTP) 허용
스팸 도구화 → ISP 차단 / IP blacklist.

### 함정 5 — Authentication 평문
HTTP proxy 면 `Proxy-Authorization: Basic ...` 평문. **HTTPS proxy** 또는 NTLM/Kerberos.

---

## 학습 자료

- **RFC 9110** Section 9.3.6 (TRACE) / 9.3.6 (CONNECT)
- **RFC 9298** (CONNECT-UDP for HTTP/3)
- "Cross-Site Tracing" — Jeremiah Grossman 2003
- Squid Proxy 문서

---

## 관련

- [[methods]] — Methods hub
- [[options]] — CORS preflight
- [[../security/security]] — XST, Open Proxy
- [[../../cdn-proxy/cdn-proxy]] — Proxy hub
