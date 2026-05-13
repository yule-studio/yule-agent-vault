---
title: "WireGuard — 모던 VPN"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T05:20:00+09:00
tags:
  - network
  - vpn
  - wireguard
---

# WireGuard — 모던 VPN

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 9412 / 핸드셰이크 / 설정 |

**[[ssh-rdp-vpn|↑ 원격 접속]]** · **[[vpn|↑↑ VPN]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **표준** | RFC 9412 (2023) — 처음 standardized |
| **개발** | Jason Donenfeld (2018) |
| **계층** | UDP (any port) |
| **코드 크기** | ~4000 줄 (kernel) |
| **암호화** | ChaCha20-Poly1305 (고정) |

---

## 1. 한 줄 정의

**모던 / 단순 / 빠른 VPN**. Linux kernel 모듈. 옛 OpenVPN / IPsec 의 복잡함 / 무거움 해결.

---

## 2. 설계 철학

### 단순
- ~4000 줄 (vs OpenVPN 수십만 줄)
- 감사 / 검증 가능
- Cipher 선택 X — 고정

### 빠름
- Kernel space (Linux)
- ChaCha20-Poly1305 — 빠르고 안전
- UDP 만

### 안전
- Static cipher — downgrade 공격 X
- Noise Protocol Framework (Trevor Perrin)
- Curve25519 / BLAKE2s / HKDF

---

## 3. 핸드셰이크

### Noise IK pattern
```
Initiator → Responder:  e, es, s, ss  (handshake_init)
Responder → Initiator:  e, ee, se      (handshake_response)
```

### 1-RTT
- 첫 패킷 — 즉시 데이터 + handshake
- 추가 round-trip X

### Key rotation
- 매 ~2 분 자동 (REKEY_AFTER_TIME)
- Forward secrecy

---

## 4. 키

### 키 생성
```bash
wg genkey | tee private.key | wg pubkey > public.key
```

### Pre-shared key (옵션)
```bash
wg genpsk > psk
```

→ Quantum-resistant 추가 보안.

---

## 5. 설정 — Server

### /etc/wireguard/wg0.conf
```ini
[Interface]
PrivateKey = <server private key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client public key>
AllowedIPs = 10.0.0.2/32

[Peer]
PublicKey = <client2 public key>
AllowedIPs = 10.0.0.3/32
```

### 시작
```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

---

## 6. 설정 — Client

### /etc/wireguard/wg0.conf
```ini
[Interface]
PrivateKey = <client private key>
Address = 10.0.0.2/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server public key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0           # 모든 트래픽 VPN
PersistentKeepalive = 25
```

### 시작
```bash
sudo wg-quick up wg0
```

### Split tunnel
```
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
```

→ 내부 망만 VPN.

---

## 7. 명령

```bash
# 상태
sudo wg                       # 모든 인터페이스
sudo wg show wg0
sudo wg show wg0 dump

# 시작 / 종료
sudo wg-quick up wg0
sudo wg-quick down wg0

# 키 생성
wg genkey
wg pubkey
wg genpsk

# 인터페이스 설정
sudo wg set wg0 peer <pubkey> allowed-ips 10.0.0.5/32
sudo wg set wg0 peer <pubkey> remove
```

---

## 8. Allowed IPs — 핵심

### 의미
- **Outgoing routing** — 어느 peer 로 라우팅
- **Incoming filter** — 어느 IP 의 패킷 허용 (Cryptokey Routing)

### 예
```
Peer A: AllowedIPs = 10.0.0.2/32
# A 의 트래픽 — 10.0.0.2 만 허용
# 10.0.0.2 행 패킷 — A 로
```

### 모든 트래픽 VPN
```
AllowedIPs = 0.0.0.0/0, ::/0
```

### 특정 망만
```
AllowedIPs = 10.0.0.0/24
```

---

## 9. NAT / Mobile

### PersistentKeepalive
```
PersistentKeepalive = 25
```

→ 25 초 마다 keep-alive — NAT mapping 유지.

### Roaming
- Mobile WiFi ↔ LTE 전환 — 자동 재연결
- 새 endpoint 자동 학습

---

## 10. WireGuard vs OpenVPN vs IPsec

| | WireGuard | OpenVPN | IPsec |
| --- | --- | --- | --- |
| **속도** | 가장 빠름 | 보통 | 빠름 |
| **레이턴시** | 낮음 | 보통 | 보통 |
| **설정** | 간단 | 복잡 | 매우 복잡 |
| **NAT** | 좋음 | 좋음 | 어려움 |
| **모바일** | 좋음 | 보통 | 좋음 (IKEv2) |
| **암호화** | ChaCha20 고정 | 다양 | 다양 |
| **표준** | RFC 9412 | de facto | RFC 4301 |
| **코드** | ~4000 | 수십만 | 매우 큼 |

### 성능 (참고)
- WireGuard: ~1 Gbps+ on commodity HW
- OpenVPN: ~300 Mbps
- IPsec (AES-NI): ~700 Mbps

---

## 11. Tailscale / Headscale

### Tailscale
- WireGuard + Coordination server
- 키 교환 / 인증 (Google / GitHub / SSO)
- NAT traversal — STUN / DERP relay

### Headscale
- Tailscale 의 self-hosted coordination
- 오픈소스

### 흐름
```
User → Tailscale account (OAuth)
Coordination server 가 키 / IP / ACL 관리
P2P 시도 (STUN)
NAT 막힌 경우 — DERP relay
```

### 효과
- 작은 회사 / 개인 — VPN 보다 쉬움
- ACL 기반 권한
- 100 device 무료

---

## 12. Mesh / P2P 토폴로지

### Hub-and-spoke
```
모든 클라이언트 → 한 서버 → 인터넷
```

### Mesh
```
각 peer 가 직접 통신 (필요 시)
Tailscale / Nebula / Netbird
```

### 장점
- Latency ↓
- 서버 부하 ↓

### 단점
- 키 / IP / ACL 관리 복잡

---

## 13. WireGuard on macOS / iOS / Windows

### macOS / iOS
- App Store — WireGuard
- Network Extension 기반

### Windows
- WireGuard for Windows (공식)
- 시스템 서비스

### Linux
- Kernel 모듈 (5.6+) — 가장 빠름
- 옛 — wireguard-go (userspace)

---

## 14. 함정

### 함정 1 — UDP 차단
일부 공용 WiFi / 회사 망 — UDP 차단.
WireGuard — TCP 옵션 없음 (의도).
우회 — tunnel over TCP (wireguard-go) / OpenVPN.

### 함정 2 — PostUp / PostDown 의 iptables
시스템 마다 다름 — NAT / forwarding 검토.

### 함정 3 — DNS leak
DNS = 10.0.0.1 — 사용자 시스템 따름.
일부 — DNS 가 VPN 우회.

### 함정 4 — Cryptokey routing 의 잘못된 ACL
AllowedIPs 의 중복 — 첫 매치만.

### 함정 5 — Keep-alive 없으면 NAT timeout
PersistentKeepalive 권장.

### 함정 6 — 키 회전 절차 없음
키 영구. 회전 운영 필요.

### 함정 7 — Static key 의 양자 위협
미래 — PQC (post-quantum) 준비. PSK 활용.

---

## 15. 학습 자료

- **RFC 9412** (WireGuard)
- "WireGuard: Next Generation Kernel Network Tunnel" (Donenfeld)
- wireguard.com docs
- "Tailscale" 블로그

---

## 16. 관련

- [[vpn]] — VPN 총론
- [[ssh-rdp-vpn]] — Hub
- [[../topics/zero-trust]]
