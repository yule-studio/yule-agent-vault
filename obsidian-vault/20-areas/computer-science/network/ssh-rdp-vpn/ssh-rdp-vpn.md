---
title: "원격 접속 — SSH / RDP / VPN"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T05:00:00+09:00
tags:
  - network
  - ssh
  - rdp
  - vpn
---

# 원격 접속 — SSH / RDP / VPN

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SSH / RDP / VPN / WireGuard hub |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-7-application/layer-7-application|↑↑ L7]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **SSH** | 22 / TCP, RFC 4251-4254 |
| **RDP** | 3389 / TCP+UDP |
| **OpenVPN** | 1194 / UDP (or TCP) |
| **WireGuard** | UDP (any port) |
| **IPsec** | 500/4500 / UDP, 50/51 (ESP/AH) |

---

## 1. 4 가지 — 한눈

| 기술 | 용도 | 계층 |
| --- | --- | --- |
| **SSH** | 원격 셸 / 파일 / 터널 | L7 (TCP 위) |
| **RDP** | Windows GUI 원격 | L7 |
| **VPN** | 네트워크 전체 터널 | L3 (IPsec) / L4 (OpenVPN/WireGuard) |
| **VNC** | 화면 픽셀 공유 (옛) | L7 |

---

## 2. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[ssh]] | SSH — 셸 / 파일 / 터널 |
| [[rdp-vnc]] | RDP / VNC — GUI |
| [[vpn]] | VPN 총론 (OpenVPN / IPsec / WireGuard) |
| [[wireguard]] | WireGuard — 모던 VPN |

---

## 3. 흐름 — Bastion / Jump host

```
Client → Bastion (SSH) → Internal server
```

### 모던 — Zero Trust
- IAP (Google), Tailscale, Cloudflare Access
- VPN 없이 — Identity 기반 게이트웨이

자세히 → [[../topics/zero-trust]]

---

## 4. SSH vs RDP

| | SSH | RDP |
| --- | --- | --- |
| **OS** | Unix / Linux / macOS | Windows |
| **인터페이스** | 텍스트 셸 | GUI |
| **인증** | 키 / password | password (옛) / Kerberos |
| **포트** | 22 | 3389 |
| **암호화** | 강함 (모던) | TLS / NLA |
| **터널** | 강력 (포트 / SOCKS / X11) | 일부 |

---

## 5. VPN — 3 가지 비교

| | OpenVPN | IPsec | WireGuard |
| --- | --- | --- | --- |
| **계층** | L4 (TCP/UDP) | L3 (IP) | L3+L4 (UDP) |
| **속도** | 보통 | 빠름 | 가장 빠름 |
| **설정** | 복잡 | 매우 복잡 | 간단 |
| **모바일 친화** | 보통 | 좋음 (IKEv2) | 좋음 |
| **코드 크기** | 수십만 줄 | 큼 | ~4000 줄 |
| **표준** | de facto | RFC 4301+ | RFC 9412 (2023) |

---

## 6. 위협 / 보안

| 위협 | 방어 |
| --- | --- |
| **Brute force (SSH)** | fail2ban / 키 기반 / port 변경 |
| **RDP exposure** | VPN 뒤 / NLA / MFA |
| **VPN 키 유출** | rotate / 짧은 cert TTL |
| **Tunnel hijacking** | 강한 인증 / MFA |
| **Side-channel** | TLS / 모던 cipher |

---

## 7. Bastion vs Zero Trust

### Bastion (옛)
- 한 게이트웨이 → 내부 SSH
- VPN + 사용자 인증

### Zero Trust (모던)
- 인증 / 권한 매번 검증
- Identity-aware proxy
- BeyondCorp (Google)

자세히 → [[../topics/zero-trust]]

---

## 8. 학습 자료

- **RFC 4251** (SSH 아키텍처)
- **RFC 4301** (IPsec)
- **RFC 9412** (WireGuard)
- "SSH, the Secure Shell" (Barrett)
- "BeyondCorp" Google paper

---

## 9. 관련

- [[../tls-ssl/tls-ssl]] — RDP / OpenVPN 의 TLS
- [[../topics/zero-trust]]
- [[../osi-7-layer/layer-7-application/layer-7-application]]
