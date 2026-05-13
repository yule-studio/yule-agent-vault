---
title: "RDP / VNC — GUI 원격 접속"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T05:10:00+09:00
tags:
  - network
  - rdp
  - vnc
  - remote
---

# RDP / VNC — GUI 원격 접속

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RDP / VNC / NLA / 비교 |

**[[ssh-rdp-vpn|↑ 원격 접속]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | RDP | VNC |
| --- | --- | --- |
| **개발** | Microsoft (1998) | AT&T Cambridge (1998) |
| **포트** | 3389 | 5900+display |
| **OS** | Windows / Linux (xrdp) | 크로스 플랫폼 |
| **인터페이스** | OS native | 픽셀 비트맵 |
| **암호화** | TLS / NLA | 약함 (대부분) |

---

## 1. RDP — Remote Desktop Protocol

### 정의
- Microsoft 의 GUI 원격 — Windows Server / Terminal Services
- ITU-T T.120 기반

### 특징
- 화면 / 키보드 / 마우스 / 오디오 / 파일 / 프린터 / clipboard
- 다중 채널 (압축 / 캐시)
- TLS 암호화 / NLA (Network Level Authentication)

### 동작
```
Client → Server: TCP 3389
TLS handshake
NLA — 자격증명 검증 (서버 자원 보호)
RDP session 시작
```

### NLA — Network Level Authentication
- 세션 시작 전에 인증
- 자원 / DoS 방지
- 권장 옵션

### 보안
- **VPN 뒤** — 인터넷 직접 노출 X
- **MFA** — Azure MFA / Duo
- **계정 잠금** — brute force 방지
- **포트 변경** — 보안 by obscurity (큰 효과 X)

### RDP 가 표적
- BlueKeep (CVE-2019-0708) — wormable
- WannaCry / Petya — RDP 약점 활용 (역사적)

---

## 2. xrdp — Linux 의 RDP 서버

```bash
sudo apt install xrdp
sudo systemctl enable xrdp
# 3389 port 에서 Linux desktop 접속
```

### 동작
- RDP protocol → X11 / Wayland 세션
- 사용자 별 desktop

---

## 3. RDP 클라이언트

| OS | 클라이언트 |
| --- | --- |
| Windows | mstsc (내장) |
| macOS | Microsoft Remote Desktop |
| Linux | Remmina, FreeRDP |
| Web | Apache Guacamole |

---

## 4. VNC — Virtual Network Computing

### 정의
- 화면의 **픽셀 데이터** 를 전송 (RFB protocol — Remote FrameBuffer)
- 1998 AT&T Cambridge → RealVNC 분기

### 특징
- 크로스 플랫폼 (Windows / Mac / Linux)
- 단순 — 픽셀 변경만 전송
- 오디오 / 파일 / clipboard 약함

### 한계
- 픽셀 기반 → 느림 (텍스트 / 그래픽 캐시 X)
- 대부분 옛 구현 — 평문 / 약한 암호화
- TLS / SSH 터널 필수

### 변종
- **RealVNC** — 원조
- **TightVNC** — 압축
- **TurboVNC** — JPEG 가속
- **UltraVNC** — Windows 특화
- **noVNC** — HTML5 / 브라우저

---

## 5. RDP vs VNC

| | RDP | VNC |
| --- | --- | --- |
| **속도** | 빠름 (캐시) | 느림 (픽셀) |
| **인증** | NLA / Kerberos | 약함 (대부분) |
| **암호화** | TLS | 옛 — 평문 |
| **다중 사용자** | Terminal Services | 한 데스크탑 |
| **오디오** | 좋음 | 약함 |
| **클립보드** | 좋음 | 가능 |
| **프린터 / 파일** | 좋음 | 약함 |

### 결론
- Windows 원격 — **RDP**
- Linux / 크로스 — **SSH X11** 또는 **xrdp**
- 임베디드 / 옛 — **VNC** (SSH 터널)

---

## 6. SSH 의 X11 forwarding (Linux GUI)

```bash
ssh -X alice@linux-server
# 또는 trusted
ssh -Y alice@linux-server

xeyes &
firefox &       # 로컬 X 에서 띄움
```

### XQuartz (macOS) / VcXsrv (Windows)
- 로컬 X server 필요

### 모던 — RDP / VNC 가 더 빠름

---

## 7. Apache Guacamole — Web 기반

### 정의
- 브라우저로 SSH / RDP / VNC / Telnet
- HTML5 — 클라이언트 불필요

### 동작
```
Browser ←─HTTP/WS─→ guacd (proxy) ←─RDP/VNC/SSH─→ Target
```

### 효과
- 클라이언트 설치 X
- BYOD / 외부 사용자
- 중앙 인증 / 로깅

---

## 8. RDP Gateway / Bastion

### RD Gateway
- Microsoft — HTTPS 위의 RDP
- 443 port — 방화벽 친화
- 외부 → Gateway (HTTPS) → 내부 RDP

### Bastion (Azure / AWS)
- 클라우드 — 관리형 Bastion
- 사용자 → 브라우저 → Bastion → VM
- VPN 불필요

---

## 9. 함정

### 함정 1 — RDP 인터넷 직접 노출
표적 1 번. VPN / Bastion 뒤로.

### 함정 2 — VNC 평문
대부분 옛 VNC — 평문. SSH 터널 / TLS.

### 함정 3 — Password 단독
MFA 필수.

### 함정 4 — 약한 RDP cipher
옛 RDP — RC4 가능. TLS 1.2 / FIPS 권장.

### 함정 5 — Session hijacking
잠금 안 한 채 자리 비움. lock 자동.

### 함정 6 — Clipboard / 파일 leak
보안 정책 — 일부 disable.

---

## 10. 학습 자료

- **MS-RDPBCGR** (RDP spec)
- **RFC 6143** (RFB / VNC)
- "Apache Guacamole" docs
- Microsoft RDS docs

---

## 11. 관련

- [[ssh-rdp-vpn]] — Hub
- [[ssh]] — SSH X11
- [[vpn]] — RDP 보호
- [[../tls-ssl/tls-ssl]] — RDP TLS
