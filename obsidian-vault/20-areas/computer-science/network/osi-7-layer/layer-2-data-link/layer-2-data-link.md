---
title: "L2 — 데이터 링크 계층 (Data Link Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:35:00+09:00
tags:
  - network
  - osi
  - layer-2
  - data-link
---

# L2 — 데이터 링크 계층 (Data Link Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Data Link 개요 + 세부 노트 9 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | **Frame (프레임)** |
| **주소 / 식별자** | **MAC 주소** (6 byte / 48 bit) |
| **대표 장치** | L2 Switch, Bridge, NIC (MAC), Wi-Fi AP, 셀룰러 기지국 |
| **대표 프로토콜** | Ethernet (IEEE 802.3), Wi-Fi (802.11), PPP, HDLC, ARP, STP, VLAN, ATM, Frame Relay, MPLS (사실 2.5) |
| **상위 계층** | L3 Network |
| **하위 계층** | L1 Physical |

---

## 1. 한 줄 정의

**물리 매체를 통해 안정적으로 프레임을 송수신** 하는 계층. **인접한 두 노드 사이
(point-to-point or shared medium)** 에서 동작.

- L1 의 비트 흐름을 **프레임으로 묶기** (Framing)
- **물리 주소 (MAC)** 로 인접 노드 식별
- **오류 검출** (CRC)
- **흐름 제어** + **매체 접근 제어 (MAC)**

---

## 2. L2 의 두 개 sublayer (IEEE 모델)

```
┌─────────────────────────────────┐
│  LLC (Logical Link Control)     │  ← IEEE 802.2 — 상위 프로토콜 식별 / 흐름·오류 제어
├─────────────────────────────────┤
│  MAC (Medium Access Control)    │  ← 매체 접근 / 프레이밍 / MAC 주소 / CRC
└─────────────────────────────────┘
            ↓ L1 Physical ↓
```

### 2.1 MAC (Medium Access Control)
- 누가 매체를 쓸 차례? (CSMA/CD, CSMA/CA, TDMA, Polling)
- 프레임 형식 + MAC 주소 + CRC
- HW (NIC) 에 구현

자세히 → [[csma-cd-ca]]

### 2.2 LLC (Logical Link Control)
- 상위 프로토콜 (IPv4 / IPv6 / ARP 등) 구분
- 흐름 / 오류 제어 (실제로는 잘 안 씀 — TCP 가 처리)
- IEEE 802.2 표준

---

## 3. L2 의 5 가지 책임

### 3.1 Framing (프레이밍)
비트 흐름을 프레임 (시작/끝 구분) 으로 묶기.

방식:
- **문자 카운트** — 헤더에 길이
- **플래그 + Byte stuffing** — `0x7E` 플래그 + 데이터 내 `0x7E` 는 escape
- **플래그 + Bit stuffing** — `01111110` 플래그 + 5 개 1 후 0 삽입
- **물리 계층 코딩 위반** — Manchester 의 invalid 패턴

자세히 → [[ethernet-frame]]

### 3.2 Addressing (주소)
- **MAC 주소** — 48 bit, NIC 출고 시 고정
- L3 (IP) 의 논리 주소와 다름

자세히 → [[mac-address]]

### 3.3 Error Detection (오류 검출)
- **CRC (FCS)** — Ethernet 의 마지막 4 byte
- Parity, Checksum 도 있지만 약함

자세히 → [[error-detection-crc]]

### 3.4 Flow Control (흐름 제어)
- Stop & Wait, Sliding Window
- 실제 Ethernet 에선 PAUSE frame (802.3x)
- TCP 가 주로 처리

자세히 → [[flow-control]]

### 3.5 Medium Access Control
- 누가 매체 점유하는가?
- CSMA/CD (옛 이더넷), CSMA/CA (Wi-Fi), TDMA (셀룰러), Polling

자세히 → [[csma-cd-ca]]

---

## 4. L2 표준 / 프로토콜 분류

### 4.1 IEEE 802 패밀리

| 표준 | 영역 |
| --- | --- |
| **802.1** | 브리징, VLAN (802.1Q), STP (802.1D), QoS |
| **802.2** | LLC |
| **802.3** | Ethernet |
| **802.5** | Token Ring (사장) |
| **802.11** | Wi-Fi |
| **802.15** | WPAN (Bluetooth, Zigbee) |
| **802.16** | WiMAX (사장) |
| **802.1X** | 인증 (Port Security) |

### 4.2 점대점 (Point-to-Point)

| 프로토콜 | 용도 |
| --- | --- |
| **PPP** (Point-to-Point) | 다이얼업 / DSL / WAN |
| **HDLC** | 시리얼 라인 / WAN |
| **SLIP** | 옛 다이얼업 |
| **PPPoE** | DSL on Ethernet |

### 4.3 WAN

| 프로토콜 | 비고 |
| --- | --- |
| **Frame Relay** | 사장 |
| **ATM** (Asynchronous Transfer Mode) | 53-byte cell, 통신사 백본 (사장) |
| **MPLS** | L2.5 — 통신사 백본 |

---

## 5. L2 Switch — 동작 원리

### 5.1 학습 (Learning)

```
Switch 가 프레임 받음
  ↓
출발 MAC 주소 + 들어온 포트를
MAC Address Table 에 저장 (CAM table)
```

### 5.2 전달 (Forwarding)

```
프레임 받음
  ↓
목적지 MAC 검색
├ 테이블에 있음   → 해당 포트로만 전송
├ 테이블에 없음   → Broadcast (모든 포트)
└ Broadcast MAC  → 모든 포트
```

### 5.3 노화 (Aging)
- 학습된 entry 5 분 (기본) 안 사용 → 삭제
- Topology 변화 적응

### 5.4 충돌 도메인 / 브로드캐스트 도메인

- **Hub**: 1 충돌 도메인 + 1 broadcast 도메인
- **Switch**: N 충돌 도메인 + 1 broadcast 도메인 (포트 별)
- **Router**: N 충돌 + N broadcast

자세히 → [[switching-bridging]]

---

## 6. 주요 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[ethernet-frame]] | Ethernet II / 802.3 / Jumbo / VLAN tag / Preamble / FCS |
| [[mac-address]] | 48 bit 구조, OUI, NIC vendor, Locally Admin |
| [[csma-cd-ca]] | CSMA/CD (옛 이더넷), CSMA/CA (Wi-Fi), ALOHA, Slotted ALOHA |
| [[error-detection-crc]] | Parity, Checksum, CRC 다항식 |
| [[flow-control]] | Stop & Wait, Sliding Window, HDLC, 802.3x PAUSE |
| [[switching-bridging]] | L2 Switch 동작, CAM, Cut-through vs Store-and-forward |
| [[vlan-stp-bonding]] | VLAN (802.1Q), STP (802.1D / RSTP / MSTP), LAG (LACP) |
| [[dte-dce-interfaces]] | DTE / DCE, RS-232/422/485, X.21 |
| [[wireless-link]] | 802.11 MAC, RTS/CTS, Bluetooth, Zigbee, NFC |

---

## 7. L2 의 함정

### 함정 1 — MAC 주소 = 영구 가정
NIC 변경, MAC spoofing, 가상 NIC (VM/Docker) 으로 변함.

### 함정 2 — Switch 가 L2 만 한다 가정
모던 L3 Switch, L4-7 Switch 도 흔함.

### 함정 3 — Broadcast 폭풍
Switch 루프 → 무한 forwarding → 네트워크 마비. STP 필수.

### 함정 4 — VLAN 의 보안 가정
VLAN hopping 공격 — 802.1X / Port Security 필요.

### 함정 5 — Wi-Fi 의 "충돌 없음" 가정
CSMA/CA 는 충돌 회피, 완전 방지 X. Hidden node 문제.

### 함정 6 — CRC 가 모든 오류 잡음
악의적 변조는 CRC 와 함께 갱신 가능 → 무결성은 L7 (HMAC).

---

## 8. 면접 / 토픽

1. **L2 vs L3 차이**.
2. **MAC 주소 vs IP 주소**.
3. **Switch 동작 원리** (learning, forwarding, flooding).
4. **CSMA/CD vs CSMA/CA**.
5. **VLAN 의 의미와 802.1Q**.
6. **STP 의 필요성**.
7. **충돌 도메인 / 브로드캐스트 도메인**.
8. **Hub vs Switch vs Router**.

---

## 9. 학습 자료

- **Computer Networks** (Tanenbaum / Wetherall) — Ch. 3-4
- **TCP/IP Illustrated Vol.1** (Stevens) — Ch. 2-3
- IEEE 802 표준 문서
- Cisco "CCNA Routing and Switching"

---

## 10. 관련

- [[../layer-1-physical/layer-1-physical]] — 하위
- [[../layer-3-network/layer-3-network]] — 상위
- [[../../routing/routing]] — L3 라우팅
- [[../osi-7-layer]] — OSI 7 hub
