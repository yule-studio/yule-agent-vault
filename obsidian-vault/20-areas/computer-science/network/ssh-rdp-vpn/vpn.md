---
title: "VPN — Virtual Private Network 총론"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T05:15:00+09:00
tags:
  - network
  - vpn
  - ipsec
  - openvpn
---

# VPN — Virtual Private Network 총론

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | IPsec / OpenVPN / 비교 |

**[[ssh-rdp-vpn|↑ 원격 접속]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **목적** | 공용 네트워크 위에 사설 터널 |
| **유형** | Site-to-Site / Remote Access |
| **계층** | L3 (IPsec) / L4 (OpenVPN) |
| **표준** | RFC 4301 (IPsec) |

---

## 1. 한 줄 정의

**공용 네트워크에 가상 사설망** — 암호화된 터널로 IP 패킷을 캡슐화.

---

## 2. VPN 유형

### Site-to-Site
- 본사 ↔ 지사 (망 ↔ 망)
- 라우터 / 방화벽이 터널 endpoint
- IPsec 가 표준

### Remote Access
- 개인 ↔ 회사 (PC ↔ 망)
- VPN 클라이언트 → VPN 서버
- OpenVPN / WireGuard / IPsec IKEv2

### SSL VPN
- HTTPS 위에 (443) — 방화벽 친화
- Cisco AnyConnect / Pulse Secure / OpenVPN
- 옛 — Java applet

---

## 3. VPN 의 3 주요 — 비교

| | OpenVPN | IPsec | WireGuard |
| --- | --- | --- | --- |
| **계층** | L4 (UDP/TCP) | L3 (IP) | L3+L4 (UDP) |
| **표준** | de facto | RFC 4301 | RFC 9412 (2023) |
| **속도** | 보통 | 빠름 | 가장 빠름 |
| **설정** | 복잡 (cert/PKI) | 매우 복잡 | 간단 (키 2개) |
| **모바일** | 보통 | 좋음 (IKEv2) | 좋음 |
| **NAT** | UDP/TCP 모두 | UDP 4500 (NAT-T) |  좋음 |
| **코드** | 수십만 줄 | 매우 큼 | ~4000 줄 |
| **암호화** | OpenSSL | 다양 | ChaCha20 |
| **검증** | TLS PKI | IKE (PSK / cert) | static key |

자세히 → [[wireguard]]

---

## 4. IPsec — IP Security

### 구성
```
IKE (Internet Key Exchange)  — 키 협상
        ↓
ESP (Encapsulating Security Payload)  — 암호화 + 무결성 (50 protocol)
AH (Authentication Header)            — 무결성 only (51 protocol)
```

### IKE 버전
- **IKEv1** (RFC 2409) — 옛
- **IKEv2** (RFC 7296) — 권장 (mobile / NAT 친화)

### Tunnel mode / Transport mode
```
Transport:    IP | ESP | TCP | data       (호스트 ↔ 호스트)
Tunnel:    IP | ESP | IP | TCP | data     (게이트웨이 ↔ 게이트웨이)
```

→ Site-to-Site 는 tunnel mode.

### Cipher
- AES-GCM (AEAD)
- ChaCha20-Poly1305

### NAT-T (RFC 3947)
- ESP 가 NAT 통과 어려움 → UDP 4500 으로 캡슐화

### 한계
- 설정 매우 복잡 (Phase 1 / Phase 2, SA, SPI, ...)
- 디버깅 어려움

---

## 5. OpenVPN

### 정의
- 2001, James Yonan
- TLS 위의 UDP / TCP 터널
- 크로스 플랫폼

### 동작
```
Client → Server: UDP 1194 (or TCP)
TLS handshake (PKI / cert)
Encrypted tunnel
```

### 구성
```
/etc/openvpn/server.conf:
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
cipher AES-256-GCM
auth SHA256
tls-auth ta.key 0
```

### 인증
- PKI (cert) — 권장
- Username / password
- 2FA — PAM plugin

### 장점
- 안정 / 검증
- 방화벽 친화 (TCP 443 옵션)
- 다양한 OS

### 단점
- 느림 (TLS overhead)
- 설정 복잡
- WireGuard 보다 무거움

---

## 6. WireGuard

자세히 → [[wireguard]]

### 요약
- 2018, Jason Donenfeld
- Linux kernel 모듈 (5.6+)
- ~4000 줄 — 감사 가능
- ChaCha20-Poly1305 / Curve25519 (고정)
- 가장 빠름

---

## 7. 클라우드 VPN — Mesh

### Tailscale
- WireGuard 기반
- Coordination server (인증 / 키 교환)
- NAT traversal — STUN / DERP relay
- P2P 가능 시 P2P, 안 되면 relay

### Nebula (Slack)
- Lighthouse + Host

### ZeroTier
- L2 over Internet
- 가상 이더넷

### Cloudflare Tunnel
- Outbound 만 — 방화벽 friendly
- Cloudflare 에지가 endpoint

### 모던
- VPN 없이 — Zero Trust (Cloudflare Access, IAP)

---

## 8. VPN 클라이언트 (Remote Access)

| 클라이언트 | 기반 |
| --- | --- |
| Cisco AnyConnect | TLS / IKEv2 |
| Pulse Secure | TLS |
| FortiClient | IPsec / SSL |
| OpenVPN Connect | OpenVPN |
| WireGuard | WireGuard |
| Tailscale | WireGuard + coordination |

---

## 9. 모바일 VPN

### IKEv2 / MOBIKE (RFC 4555)
- 네트워크 전환 시 (WiFi ↔ LTE) — 세션 유지
- 모바일 친화

### WireGuard
- UDP 손실 / NAT — 재연결 빠름
- Battery — 적은 keep-alive

---

## 10. 함정

### 함정 1 — VPN 의 DNS leak
DNS 가 VPN 우회. push DNS / Block-Outside-DNS.

### 함정 2 — Split tunnel
일부 트래픽만 VPN — 정책 필요.

### 함정 3 — PSK (Preshared Key)
유출 시 모두 위협. 회전.

### 함정 4 — Client cert 회전 X
TTL 짧게 / CA 운영.

### 함정 5 — NAT 의 ESP
IPsec — UDP 4500 (NAT-T) 사용.

### 함정 6 — 인터넷 속도 저하
VPN 의 overhead — 10-30%.

### 함정 7 — VPN 의 "anonymous"
공급자 로그 가능. 무료 VPN — spyware 사례.

---

## 11. 학습 자료

- **RFC 4301** (IPsec)
- **RFC 7296** (IKEv2)
- "OpenVPN" docs
- "WireGuard" 사이트
- "Tailscale" docs

---

## 12. 관련

- [[ssh-rdp-vpn]] — Hub
- [[wireguard]] — 모던 VPN
- [[../tls-ssl/tls-ssl]] — OpenVPN TLS
- [[../topics/zero-trust]] — VPN 대체
