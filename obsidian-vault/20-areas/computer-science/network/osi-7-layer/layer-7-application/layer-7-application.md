---
title: "L7 — 응용 계층 (Application Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:15:00+09:00
tags:
  - network
  - osi
  - layer-7
  - application
---

# L7 — 응용 계층 (Application Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | L7 hub + 응용 프로토콜 폴더 인덱스 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

> 깊이는 각 응용 프로토콜 폴더 — [[../../http/http]], [[../../dns/dns]],
> [[../../mail/mail]], [[../../ssh-rdp-vpn/ssh-rdp-vpn]] 등.

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | message |
| **주소 / 식별자** | URL / URI / 도메인 이름 / 이메일 주소 |
| **대표 장치** | L7 Switch (Application LB, WAF), 프록시 |
| **대표 프로토콜** | **HTTP**, **DNS**, **SMTP/POP3/IMAP**, **SSH**, **FTP/SFTP**, RDP, Telnet, SNMP, NTP, LDAP, NFS, SMB |

---

## 1. 한 줄 정의

**사용자 / 응용이 직접 다루는 프로토콜** — 웹, 이메일, 파일 전송, 원격 접속 등.
OSI 의 가장 위, 사용자에게 가장 가까운 계층.

---

## 2. 카테고리별 응용 프로토콜

### 2.1 웹 / API

| 프로토콜 | 포트 | 폴더 |
| --- | --- | --- |
| HTTP/1.1 | 80 | [[../../http/http|↗ http]] |
| HTTP/2 | 443 (TLS) | [[../../http/http]] |
| HTTP/3 (QUIC) | 443 (UDP) | [[../../http/http]] |
| WebSocket | 80/443 | [[../../http/http]] |
| WebRTC | UDP | [[../../rpc-messaging/rpc-messaging]] |
| gRPC | 443 | [[../../rpc-messaging/rpc-messaging]] |
| GraphQL | 443 | [[../../rpc-messaging/rpc-messaging]] |

### 2.2 이름 해결

| 프로토콜 | 포트 | 폴더 |
| --- | --- | --- |
| **DNS** | 53 | [[../../dns/dns|↗ dns]] |
| mDNS | 5353 | [[../../dns/dns]] |
| LLMNR | 5355 | — |
| DoH / DoT | 443 / 853 | [[../../dns/dns]] |

### 2.3 메일

| 프로토콜 | 포트 | 용도 |
| --- | --- | --- |
| **SMTP** | 25, 587, 465 | 송신 |
| **POP3** | 110, 995 | 수신 (다운로드) |
| **IMAP** | 143, 993 | 수신 (동기화) |
| MAPI / EWS | — | Exchange |

자세히 → [[../../mail/mail|↗ mail]]

### 2.4 파일 전송

| 프로토콜 | 포트 |
| --- | --- |
| FTP | 21 (control) + 20 (data) |
| **SFTP** | 22 (SSH 위) |
| FTPS | 990 (TLS) |
| TFTP | 69 (UDP) |
| HTTP / WebDAV | 80/443 |
| **SMB / CIFS** | 445 |
| **NFS** | 2049 |
| rsync | 873 |
| BitTorrent | P2P |

### 2.5 원격 접속

| 프로토콜 | 포트 |
| --- | --- |
| **SSH** | 22 |
| Telnet | 23 (사장) |
| RDP | 3389 |
| VNC | 5900+ |
| X11 | 6000+ (사장) |

자세히 → [[../../ssh-rdp-vpn/ssh-rdp-vpn]]

### 2.6 디렉터리 / 인증

| 프로토콜 | 포트 |
| --- | --- |
| **LDAP** | 389 |
| LDAPS | 636 |
| Kerberos | 88 |
| RADIUS | 1812 / 1813 |
| TACACS+ | 49 |

### 2.7 시간 / 관리

| 프로토콜 | 포트 |
| --- | --- |
| **NTP** | 123 (UDP) |
| SNMP | 161 / 162 (UDP) |
| Syslog | 514 (UDP) |
| IPMI | 623 (UDP) |

### 2.8 메시징 / 큐

| 프로토콜 | 포트 | 영역 |
| --- | --- | --- |
| AMQP (RabbitMQ) | 5672 | [[../../rpc-messaging/rpc-messaging]] |
| MQTT | 1883 / 8883 (TLS) | IoT |
| STOMP | 61613 | — |
| Kafka | 9092 | — |

### 2.9 데이터베이스

| DB | 포트 |
| --- | --- |
| MySQL | 3306 |
| PostgreSQL | 5432 |
| MSSQL | 1433 |
| Oracle | 1521 |
| MongoDB | 27017 |
| Redis | 6379 |
| Cassandra | 9042 |
| Elasticsearch | 9200 |

### 2.10 게임 / 미디어

| 프로토콜 | 비고 |
| --- | --- |
| RTP / RTCP | 미디어 스트리밍 |
| RTMP | Adobe Flash 영상 |
| WebRTC | P2P 영상 / 음성 |
| SIP | VoIP signaling |
| QUIC / HTTP/3 | 게임 latency |

---

## 3. URL / URI / URN

### 3.1 URI = URL + URN

```
URI (Uniform Resource Identifier)
├── URL (Locator) — 위치 기반
└── URN (Name) — 이름 기반 (예: urn:isbn:0451450523)
```

### 3.2 URL 구조

```
scheme://[user:pass@]host[:port]/path?query#fragment

예:
https://user:pass@api.example.com:443/v1/users?id=5#info
└─┬─┘   └─────┬─────┘ └─────┬──────┘ └─┬┘ └──┬──┘ └─┬┘ └┬─┘
 scheme  userinfo         host        port  path   query fragment
```

### 3.3 주요 scheme

| Scheme | 의미 |
| --- | --- |
| `http://`, `https://` | 웹 |
| `ftp://`, `sftp://` | 파일 |
| `mailto:` | 이메일 |
| `tel:`, `sms:` | 전화 |
| `file://` | 로컬 파일 |
| `data:` | inline data (base64) |
| `ws://`, `wss://` | WebSocket |
| `git://`, `ssh://` | 코드 |
| `magnet:` | BitTorrent |

---

## 4. L7 Switch / Application LB

L4 보다 더 — 응용 헤더 / URL 보고 라우팅:

- **Path-based** — `/api/*` → API server, `/static/*` → CDN
- **Host-based** — `app.example.com` vs `admin.example.com`
- **Header-based** — `User-Agent` 모바일 / 데스크톱
- **Cookie-based** — Sticky session

자세히 → [[../../load-balancing/load-balancing]]

---

## 5. WAF (Web Application Firewall)

L7 보안:
- SQL Injection, XSS 패턴 매칭
- OWASP CRS (Core Rule Set)
- Cloudflare WAF, AWS WAF, ModSecurity

자세히 → [[../../../security-theory/security-theory]]

---

## 6. API 스타일

| 스타일 | 특징 |
| --- | --- |
| **REST** | 자원 + HTTP 메서드, 무상태 |
| **SOAP** | XML, 무겁다, 사장 |
| **GraphQL** | 단일 엔드포인트, 쿼리 |
| **gRPC** | HTTP/2 + Protobuf, 빠름 |
| **WebSocket** | 양방향 실시간 |
| **Server-Sent Events** | 서버 → 클라이언트 단방향 |
| **WebHooks** | 이벤트 콜백 |

---

## 7. 응용 프로토콜의 진화

| 시대 | 특징 |
| --- | --- |
| 1970s | Telnet, FTP — 텍스트 명령 |
| 1980s | SMTP, NNTP, DNS |
| 1990s | HTTP/1.0, SSL — 웹 시작 |
| 2000s | HTTP/1.1 보편, REST |
| 2010s | HTTPS 표준, HTTP/2, gRPC, WebSocket |
| 2020s | HTTP/3 (QUIC), GraphQL 확산 |

---

## 8. 함정

### 함정 1 — Telnet 사용
평문 — 도청 가능. SSH.

### 함정 2 — 비표준 포트의 가정
80 = HTTP 라 가정 X — 임의 포트에 임의 프로토콜.

### 함정 3 — DNS 의존
DNS 실패 시 응용 다운. 캐시 / 다중 resolver.

### 함정 4 — 시간 동기 무시
NTP 안 켠 호스트 — 인증서 검증 / Kerberos 등 실패.

### 함정 5 — 응용 protocol 의 보안
FTP / POP3 평문 — TLS 버전 사용.

### 함정 6 — SNMP v1/v2c
community string 평문. SNMPv3.

---

## 9. 학습 자료

- IANA Service Name & Port Registry
- **TCP/IP Illustrated Vol.1** Ch. 25+ (응용)
- **HTTP: The Definitive Guide** (Gourley)
- **DNS and BIND** (Liu / Albitz)

---

## 10. 관련

- [[layer-6-presentation]] — 하위
- [[../../http/http|↗ HTTP 폴더]]
- [[../../dns/dns|↗ DNS 폴더]]
- [[../../mail/mail|↗ 메일 폴더]]
- [[../../ssh-rdp-vpn/ssh-rdp-vpn|↗ SSH/RDP/VPN 폴더]]
- [[../../rpc-messaging/rpc-messaging|↗ RPC/Messaging 폴더]]
- [[../osi-7-layer]] — OSI hub
