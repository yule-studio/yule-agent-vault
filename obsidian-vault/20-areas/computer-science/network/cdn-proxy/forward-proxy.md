---
title: "Forward Proxy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T07:10:00+09:00
tags:
  - network
  - proxy
  - forward-proxy
  - squid
---

# Forward Proxy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Squid / SOCKS / Tor / 회사 proxy |

**[[cdn-proxy|↑ CDN/Proxy]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**클라이언트 측 중간자** — 클라이언트가 명시적으로 proxy 를 지정하고 모든 트래픽이 통과. 회사 / 학교 / 익명.

---

## 2. 동작

```
Client → Proxy: GET http://example.com HTTP/1.1
                Host: example.com
                ...
Proxy → Server: GET / HTTP/1.1
                Host: example.com
                ...
Proxy ← Server: 200 OK
Client ← Proxy: 200 OK
```

### 핵심
- 클라이언트가 proxy 알고 사용
- Proxy 가 서버 측에 client 의 IP / 정보 노출 X (대부분)

---

## 3. 유형

### HTTP Proxy
- HTTP / HTTPS 만 (CONNECT for HTTPS)
- 헤더 검사 / 변환

### SOCKS (RFC 1928)
- L4 — TCP / UDP 임의
- SSH dynamic forward 의 기반
- SOCKS4 / SOCKS5

### Transparent Proxy
- 클라이언트가 모름 (자동 인터셉트)
- iptables / WCCP 로 라우팅
- TLS 통과 어려움 (인증서 문제)

### Anonymous Proxy
- 자기 신원 / client IP 숨김
- Tor / proxy chain

---

## 4. HTTPS — CONNECT 방식

```
Client → Proxy: CONNECT example.com:443 HTTP/1.1

Proxy → Server: TCP connect
Proxy ← Server: TCP connected
Client ← Proxy: HTTP/1.1 200 Connection established

(이후 — TLS handshake / 데이터 — proxy 통과만)
```

### 효과
- Proxy 가 TLS 의 내용 못 봄 (단순 TCP tunnel)
- 헤더 검사 X

### MITM Proxy
- 회사 — 자체 CA 사설 cert
- Proxy 가 TLS 종료 / 재암호화
- 트래픽 검사
- 사용자 — 인증서 신뢰 설치

---

## 5. Squid — 가장 옛 / 표준

### 정의
- 1996 — Harvest project 의 분리
- HTTP / HTTPS / FTP

### 설정
```
acl allowed src 192.168.0.0/24
http_access allow allowed
http_access deny all
http_port 3128

cache_dir ufs /var/cache/squid 10000 16 256
maximum_object_size 1024 MB
```

### 사용
- 옛 — 회사 / ISP / 학교
- 모던 — Cloudflare gateway / Zscaler 가 대체

---

## 6. SOCKS Proxy

### SOCKS5 (RFC 1928)
- TCP / UDP 임의
- 인증 (none / password / GSSAPI)

### 사용
- SSH dynamic forward (`ssh -D 1080`)
- 게임 / IRC 등 임의 프로토콜
- Tor 의 데이터 plane

### 클라이언트
- 브라우저 — SOCKS5 설정
- proxychains — 명령어 wrapper

---

## 7. Tor — Onion Routing

### 정의
- 3 layer (Entry / Middle / Exit) 의 양파 라우팅
- 각 노드 — 다음 hop 만 앎
- 익명 / censorship 우회

### 흐름
```
Client → Entry → Middle → Exit → 서버
            (각 layer — 자기 layer 만 복호화)
```

### Onion Service (.onion)
- 서버도 익명
- Rendezvous point 로 만남

### 한계
- 느림 (수십 hops)
- Exit node 의 traffic 보기 가능 (HTTPS 권장)

---

## 8. PAC — Proxy Auto-Config

### 정의
- JavaScript 함수 — URL 별 proxy 결정

```javascript
function FindProxyForURL(url, host) {
    if (shExpMatch(host, "*.internal.com"))
        return "DIRECT";
    if (isInNet(host, "10.0.0.0", "255.0.0.0"))
        return "DIRECT";
    return "PROXY proxy.example.com:8080";
}
```

### WPAD
- DHCP / DNS 가 PAC URL 제공
- 자동 발견

---

## 9. 사용 — 시나리오

### 회사 / 학교
- 컨텐츠 필터 (성인 / 도박 / 게임)
- 트래픽 로깅 (감사)
- DLP (Data Loss Prevention)
- 캐시 (옛 — 대역 절약)

### 익명 / 우회
- Tor / VPN
- Public proxy (보안 위험)
- 지리 차단 우회

### 개발 / 디버깅
- Charles Proxy / mitmproxy / Fiddler
- HTTP 트래픽 가로채기 / 수정

### Tor / Censorship
- 차단 우회

---

## 10. mitmproxy — 개발자 도구

### 정의
- Python 기반 인터셉트 proxy
- HTTP / HTTPS / WebSocket / TCP

### 사용
```bash
mitmproxy --listen-port 8080
# 클라이언트 — proxy 8080 설정
# CA cert 신뢰 (mitm.it)
```

### 기능
- 요청 / 응답 가로채기
- 변조 / replay
- Python 스크립트

### 비슷
- Charles Proxy (GUI)
- Fiddler (Windows)
- BurpSuite (보안 테스트)

---

## 11. Zero Trust 의 등장

### 옛 — VPN + Proxy
- 회사 망 안에서 자유, 밖에서 막힘

### 모던 — Zero Trust
- 모든 요청 — Identity 검증
- Cloudflare Access / Google IAP / Zscaler
- VPN 없이 — proxy 가 인증 강제

자세히 → [[../topics/zero-trust]]

---

## 12. 함정

### 함정 1 — TLS 의 검사
HTTPS — MITM CA 필요. 사용자 동의 / 인증서 배포.

### 함정 2 — Public proxy
인터넷의 무료 proxy — 거의 spyware. 절대 X.

### 함정 3 — Proxy 의 ban
사이트 가 proxy IP 차단. proxy chain / residential proxy.

### 함정 4 — DNS leak
브라우저는 proxy 사용해도 — DNS 는 OS resolver 사용 가능. DoH / proxy 의 DNS.

### 함정 5 — Tor 의 exit node
Exit 가 traffic 본다. HTTPS 권장. Login 의 cookie 보호.

### 함정 6 — WebRTC IP leak
브라우저 의 WebRTC — proxy 우회해서 real IP 노출. 비활성화.

---

## 13. 학습 자료

- "Squid" docs
- "Tor Design" paper (Dingledine)
- mitmproxy / Charles / Burp docs
- RFC 1928 (SOCKS5)

---

## 14. 관련

- [[cdn-proxy]] — Hub
- [[reverse-proxy]] — 반대 개념
- [[../ssh-rdp-vpn/ssh]] — SSH -D dynamic forward
- [[../topics/zero-trust]]
