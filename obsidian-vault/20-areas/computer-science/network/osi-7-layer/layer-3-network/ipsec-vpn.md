---
title: "IPsec & VPN (GRE / VXLAN / WireGuard)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:55:00+09:00
tags:
  - network
  - layer-3
  - ipsec
  - vpn
  - vxlan
  - wireguard
---

# IPsec & VPN (GRE / VXLAN / WireGuard)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | IPsec / GRE / VXLAN / IP-in-IP / WireGuard / OpenVPN |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 한 줄 정의

**IP 패킷을 다른 IP 패킷 / 암호화된 패킷으로 캡슐화** 해 보안 / 가상 네트워크를
만드는 기술. IPsec, GRE, VXLAN, WireGuard 등.

---

## 2. VPN 종류 분류

| 종류 | 계층 | 예 |
| --- | --- | --- |
| **L2 VPN** | L2 over L3 | L2TP, VXLAN, EVPN |
| **L3 VPN** | L3 over L3 | IPsec, GRE, MPLS L3VPN |
| **L7 VPN** | TLS 기반 | OpenVPN, WireGuard (UDP+crypto), SSL VPN |

---

## 3. IPsec — IP Security (RFC 4301)

### 3.1 두 모드

#### Transport Mode
- IP 헤더 그대로, 페이로드만 암호화
- 종단 ↔ 종단 (Host-to-Host)

```
[Original IP][ESP Header][TCP/UDP][Data][ESP Trailer][ESP Auth]
```

#### Tunnel Mode (가장 흔함)
- 원본 IP 패킷 전체를 캡슐화 → 새 IP 헤더
- 게이트웨이 ↔ 게이트웨이 (Site-to-Site VPN)

```
[New IP][ESP Header][Original IP][TCP/UDP][Data][ESP Trailer][ESP Auth]
```

### 3.2 두 프로토콜

| 프로토콜 | 기능 |
| --- | --- |
| **AH** (Authentication Header) | 무결성 + 인증, 암호화 X (거의 안 씀) |
| **ESP** (Encapsulating Security Payload) | 암호화 + 무결성 (보편) |

### 3.3 IKE (Internet Key Exchange)

키 교환 + SA (Security Association) 협상.

#### IKEv1 (RFC 2409, 1998)
- Main Mode (6 메시지) / Aggressive Mode (3 메시지)
- 복잡, 약한 옵션 존재

#### IKEv2 (RFC 7296, 2014)
- 4 메시지로 협상
- EAP / Mobility / NAT-T
- 모던 표준

### 3.4 SA (Security Association)

```
SA = {SPI, dst IP, protocol(AH/ESP)}
```

각 SA 가 키 / 알고리즘 / lifetime 보유. **SAD** (SA Database) 에 저장.

### 3.5 알고리즘

| 분류 | 옵션 |
| --- | --- |
| **암호화** | AES-128/256-GCM (권장), ChaCha20, ~~3DES~~, ~~DES~~ |
| **무결성** | HMAC-SHA-256, AES-GMAC, ~~MD5~~, ~~SHA-1~~ |
| **DH 그룹** | DH-14 (2048), DH-19 (P-256 ECC), DH-20 (P-384) |
| **PRF** | SHA-256 |

### 3.6 NAT Traversal (NAT-T)
- ESP 는 NAT 통과 어려움 (포트 X)
- UDP 4500 으로 캡슐화 → NAT 통과
- 자동 감지

### 3.7 IPsec VPN 구축 예

```
Site A (10.1.0.0/24) ─── Gateway A ════ IPsec ════ Gateway B ─── Site B (10.2.0.0/24)
                              (Internet)

Tunnel Mode, ESP:
1. 10.1.0.5 → 10.2.0.10 패킷 생성
2. Gateway A 가 받음
3. Encrypt + 새 IP 헤더 (Gateway A 외부 IP → Gateway B 외부 IP) + ESP
4. Internet 통과
5. Gateway B 가 받음, decrypt
6. 10.1.0.5 → 10.2.0.10 (원본) 복원
7. Site B 로 전달
```

### 3.8 IPsec 도구

- **strongSwan** — Linux, IKEv1/v2
- **Libreswan** — IKEv2
- **OpenSwan** (사장 → strongSwan)
- Cisco ASA / Juniper / pfSense
- macOS / iOS / Windows 기본 클라이언트

---

## 4. GRE — Generic Routing Encapsulation (RFC 2784)

### 4.1 개요
- 임의 L3 프로토콜을 IP 위에 캡슐화
- 암호화 X — 보안 필요시 GRE over IPsec
- 단순, 라우팅 프로토콜 (OSPF/BGP) 통과에 흔함

### 4.2 헤더

```
[New IP (Protocol=47)][GRE Header (4-16 byte)][Original Packet]

GRE Header:
[Flags + Version (2)][Protocol Type (2)]
선택적: [Checksum (2)][Key (4)][Sequence (4)]
```

### 4.3 사용
- IPv6-over-IPv4 (6in4)
- Cisco DMVPN (NHRP + GRE + IPsec)
- 클라우드 (AWS VGW, Azure VPN Gateway)
- OpenVPN 의 `--proto gre`

### 4.4 GRE vs IP-in-IP

| 측면 | GRE | IP-in-IP |
| --- | --- | --- |
| RFC | 2784 | 2003 |
| Protocol | 47 | 4 |
| 헤더 | 4-16 byte | 0 (그냥 IP 위 IP) |
| L3 지원 | 임의 (Multi-Protocol) | IP 만 |
| 사용 | 일반 | 단순 (Linux IP-in-IP) |

---

## 5. L2TP — Layer 2 Tunneling Protocol (RFC 3931)

- L2 (PPP frame) 를 UDP 위에 캡슐화
- 자체 암호화 X → 보통 **L2TP/IPsec** 결합
- Windows / macOS 기본 클라이언트
- UDP 1701

---

## 6. VXLAN — Virtual Extensible LAN (RFC 7348)

### 6.1 개요
- L2 over L3 — 데이터센터 표준
- **24-bit VNI** (VXLAN Network Identifier) — VLAN 의 12-bit (4094) 한계 초월 (16M 네트워크)
- UDP 4789

### 6.2 헤더

```
[Outer Eth][Outer IP][Outer UDP (4789)][VXLAN Header (8)][Inner Eth Frame]

VXLAN Header:
[Flags (8)][Reserved (24)][VNI (24)][Reserved (8)]
```

### 6.3 VTEP (VXLAN Tunnel End Point)
- VXLAN 시작 / 종료 지점
- 보통 ToR Switch 또는 hypervisor (Linux bridge, OVS)

### 6.4 사용
- AWS VPC (내부)
- Google Cloud, Azure
- VMware NSX
- Kubernetes (Flannel VXLAN backend)
- OpenStack Neutron

### 6.5 MTU 영향
- 50 byte 추가 (Outer Eth 14 + IP 20 + UDP 8 + VXLAN 8)
- Underlay 가 1500 면 Overlay MTU 1450

### 6.6 EVPN (Ethernet VPN, RFC 8365)
- VXLAN + BGP control plane
- 데이터센터 / 통신사 표준

---

## 7. Geneve (RFC 8926)

- VXLAN 의 후속
- 확장 가능한 옵션 헤더
- AWS Gateway Load Balancer, NSX-T

---

## 8. NVGRE (RFC 7637)
- Microsoft GRE 변형 — Hyper-V
- VXLAN 의 경쟁자였으나 사장

## 9. STT (Stateless Transport Tunneling)
- TCP segmentation offload 활용
- Nicira (VMware) — 사장

---

## 10. WireGuard — 모던 VPN

### 10.1 개요
- Jason A. Donenfeld 2017
- 4000 줄 코드 — IPsec/OpenVPN 의 1/10
- Linux 5.6+ 커널 모듈
- UDP, 정적 포트
- ChaCha20-Poly1305 / Curve25519 / BLAKE2 — 고정

### 10.2 키 교환
- Noise Protocol Framework
- 공개키 인증 — SSH 와 비슷
- 1-RTT handshake

### 10.3 설정 예

```ini
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.1/24
PrivateKey = <generated>
ListenPort = 51820

[Peer]
PublicKey = <peer pubkey>
AllowedIPs = 10.0.0.2/32
Endpoint = peer.example.com:51820
PersistentKeepalive = 25
```

### 10.4 vs IPsec

| 측면 | IPsec | WireGuard |
| --- | --- | --- |
| 코드 | 70K+ 줄 | 4K 줄 |
| 알고리즘 | 협상 가능 | 고정 |
| 키 교환 | IKE | Noise |
| 설정 | 복잡 | 단순 |
| 성능 | 양호 | 우수 |
| Roaming | 어려움 | 좋음 |
| 표준화 | RFC | RFC 8439 (ChaCha20) — 표준 없음 |

### 10.5 사용
- Tailscale (WireGuard 기반 P2P mesh)
- Mullvad / IVPN
- 모바일 VPN

---

## 11. OpenVPN

### 11.1 개요
- TLS 기반 (OpenSSL)
- TCP 또는 UDP (UDP 권장)
- TUN (L3) / TAP (L2) 인터페이스
- 인증서 / PSK / OTP

### 11.2 vs WireGuard
- 설정 풍부 (스플릿 터널, 압축)
- 느림 (사용자 공간 처리)
- 호환성 좋음 (Windows / macOS 클라이언트)

---

## 12. SSL VPN / Clientless VPN

- HTTPS (443) 로 통과 → 방화벽 통과 잘 됨
- 브라우저 또는 클라이언트
- Cisco AnyConnect, Fortinet, Pulse Secure
- Zero Trust 와 결합

---

## 13. SD-WAN

- 여러 WAN 링크 (MPLS / Internet / LTE) 통합
- Application-aware 라우팅
- 암호화 자동 (IPsec 위)
- Vendor: VMware VeloCloud, Cisco Viptela, Fortinet, Versa

---

## 14. 함정

### 함정 1 — IPsec 의 MTU
ESP 헤더 추가로 MTU 감소 — 1400 권장. MSS Clamping 필수.

### 함정 2 — GRE 의 무보안
GRE 만으로는 도청 가능. IPsec 결합.

### 함정 3 — VXLAN underlay MTU
50 byte 오버헤드 무시 → 단편화 / 폐기.

### 함정 4 — WireGuard 의 NAT keepalive
PersistentKeepalive 안 켜면 NAT timeout. 25s 권장.

### 함정 5 — Split Tunnel vs Full Tunnel
Full Tunnel = 모든 트래픽 VPN — 회사 검증 좋지만 사용자 경험 ↓.
Split = 회사 IP 만 VPN, 나머지 직접 — 빠르지만 보안 약함.

### 함정 6 — IKEv1 사용
deprecated. IKEv2.

### 함정 7 — Open VPN concentrator
WAN edge 만 VPN, 내부는 평문 — 데이터센터 내부 도청 가능. mTLS 추가.

---

## 15. 학습 자료

- RFC 4301 (IPsec), RFC 7296 (IKEv2), RFC 7348 (VXLAN)
- WireGuard whitepaper — https://www.wireguard.com/papers/wireguard.pdf
- **IPSec, Second Edition** (Naganand Doraswamy)
- Cisco "IPsec Tunnel Mode and Transport Mode"

---

## 16. 관련

- [[../../security-theory/security-theory|↗ security-theory]] — 암호학
- [[fragmentation-mtu]] — VPN 의 MTU 영향
- [[../../tls-ssl/tls-ssl]] — OpenVPN / SSL VPN 의 기반
- [[layer-3-network]] — 상위
