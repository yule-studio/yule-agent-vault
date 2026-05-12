---
title: "MAC 주소 (Media Access Control Address)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:45:00+09:00
tags:
  - network
  - layer-2
  - mac-address
  - oui
---

# MAC 주소 (Media Access Control Address)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 48 bit 구조 / OUI / Locally Admin / Spoofing |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 한 줄 정의

**네트워크 인터페이스 (NIC) 의 물리적 주소** — 48 bit / 6 byte. IEEE 가
출고 시 부여 (대부분). 같은 LAN 안의 노드들이 서로를 구분하는 L2 식별자.

---

## 2. 표기법

```
6 byte / 48 bit, 16 진수 표기
```

| 표기 | 예 |
| --- | --- |
| **콜론** (UNIX/Linux/macOS, IEEE) | `00:1A:2B:3C:4D:5E` |
| **하이픈** (Windows) | `00-1A-2B-3C-4D-5E` |
| **Cisco** (도트) | `001a.2b3c.4d5e` |
| **연속** | `001A2B3C4D5E` |

---

## 3. MAC 주소 구조

```
┌─────────────────────────┬──────────────────────────┐
│  OUI (24 bit / 3 byte)  │ NIC-specific (24 bit)    │
│  IEEE 가 vendor 에 할당 │ vendor 가 NIC 별 부여     │
└─────────────────────────┴──────────────────────────┘

첫 옥텟의 마지막 2 비트:
  bit 0 (I/G):  0 = Unicast,    1 = Multicast
  bit 1 (U/L):  0 = Universal,  1 = Locally Administered
```

### 3.1 OUI (Organizationally Unique Identifier)
- 첫 3 byte
- IEEE 가 vendor 에 할당 (구매)
- 예: Apple `00:1B:63:xx:xx:xx`, Cisco `00:1A:2B:xx:xx:xx`, Intel `00:50:56:xx:xx:xx`

### 3.2 NIC-specific 부분
- 마지막 3 byte
- Vendor 가 자기 NIC 마다 다르게

→ 약 **281조 (2⁴⁸) 개** MAC 주소 — 사실상 무한.

### 3.3 OUI 조회
- IEEE 공식 — https://standards-oui.ieee.org/oui/oui.txt
- Wireshark, `arp -a` 자동 표시
- `00:50:56` → VMware, `00:1C:42` → Parallels, `B8:27:EB` → Raspberry Pi

---

## 4. 첫 옥텟 비트 의미

첫 옥텟 (예: `0x02`) 의 LSB 2 비트:

| Bit 0 (I/G) | Bit 1 (U/L) | 의미 |
| --- | --- | --- |
| 0 | 0 | Unicast + Universal (보통 NIC) |
| 1 | 0 | Multicast + Universal |
| 0 | 1 | Unicast + Locally Admin |
| 1 | 1 | Multicast + Locally Admin |

### 4.1 Unicast vs Multicast vs Broadcast

| 종류 | 첫 옥텟 | 예 |
| --- | --- | --- |
| **Unicast** | `xxxxxxx0` | 단일 NIC |
| **Multicast** | `xxxxxxx1` | 그룹 (예: IPv4 mcast `01:00:5E:xx:xx:xx`) |
| **Broadcast** | `FF:FF:FF:FF:FF:FF` | 모든 호스트 |

### 4.2 Multicast MAC 매핑

| IP Multicast (예) | 매핑 MAC |
| --- | --- |
| `224.0.0.1` (All Hosts) | `01:00:5E:00:00:01` |
| `224.0.0.2` (All Routers) | `01:00:5E:00:00:02` |
| `224.0.0.5` (OSPF) | `01:00:5E:00:00:05` |
| `224.0.0.9` (RIPv2) | `01:00:5E:00:00:09` |

IPv4 mcast 32 bit 의 하위 23 bit 가 MAC 의 하위 23 bit 와 매핑.

### 4.3 IPv6 Multicast

`33:33:xx:xx:xx:xx` — 마지막 32 bit 는 IPv6 의 마지막 32 bit.

---

## 5. 특수 MAC 주소

| MAC | 의미 |
| --- | --- |
| `FF:FF:FF:FF:FF:FF` | Broadcast |
| `01:80:C2:00:00:00` | STP (BPDU) |
| `01:80:C2:00:00:0E` | LLDP |
| `01:80:C2:00:00:21` | EAPOL (802.1X) |
| `00:00:5E:00:01:xx` | VRRP |
| `00:00:0C:07:AC:xx` | HSRP (Cisco) |

---

## 6. Locally Administered MAC

- 첫 옥텟의 U/L bit = 1
- vendor 가 부여한 게 아닌 OS / 사용자가 설정
- VM (VMware `00:50:56:xx:xx:xx`), Docker, Bluetooth

### 6.1 가상 MAC

- **VMware**: `00:50:56:xx:xx:xx` (Universal OUI 인 듯하지만 가상 NIC)
- **VirtualBox**: `08:00:27:xx:xx:xx`
- **Hyper-V**: `00:15:5D:xx:xx:xx`
- **Docker**: 동적 생성 (Locally Admin)
- **iOS / Android Wi-Fi 무작위화**: 프라이버시 (네트워크 별 다른 MAC)

### 6.2 Wi-Fi MAC Randomization

- iOS 14+ / Android 10+ / Windows 10+ — Wi-Fi 연결 시 랜덤 MAC
- Locally Admin bit = 1
- 추적 방지 / 프라이버시
- 일부 기업 / 학교 인증 시스템에선 비활성화

---

## 7. MAC 주소 확인 명령

### macOS / Linux
```bash
ifconfig | grep ether        # macOS
ip link                       # Linux (모던)
ip link show eth0
cat /sys/class/net/eth0/address   # Linux 직접
```

### Windows
```cmd
ipconfig /all
getmac /v
```

### ARP 캐시 (다른 호스트 MAC)
```bash
arp -a                        # macOS / Linux
ip neigh show                 # Linux 모던
```

---

## 8. MAC 주소 변경 (Spoofing)

### Linux
```bash
sudo ip link set eth0 down
sudo ip link set eth0 address 02:11:22:33:44:55
sudo ip link set eth0 up
```

### macOS (Wi-Fi)
```bash
sudo ifconfig en0 ether 02:11:22:33:44:55
```

### Windows
- 장치 관리자 → 어댑터 속성 → 고급 → Network Address

### 합법적 사용
- 프라이버시 (공공 Wi-Fi)
- 모뎀이 MAC 등록한 경우 새 라우터에 복제
- 테스트 / 디버깅

### 악의적 사용
- MAC 인증 우회 (포트 보안)
- 다른 호스트로 위장 (MITM)
- Wi-Fi 호스트 통제 회피

→ **L2 인증은 약함**. 802.1X / mTLS 필요.

---

## 9. ARP — IP ↔ MAC

ARP (Address Resolution Protocol) 는 L2 와 L3 의 다리:

```
"192.168.1.5 의 MAC 누구?"  → Broadcast (FF:FF:FF:FF:FF:FF)
"내가 그 IP — 내 MAC 은 00:1A:2B:..." → Reply

→ ARP cache 에 (192.168.1.5, 00:1A:2B:...) 저장
```

자세히 → [[../layer-3-network/arp]]

---

## 10. MAC 주소가 충돌하면?

- 같은 LAN 에 같은 MAC → ARP 충돌, 통신 불가
- 매우 드물지만 발생 가능 (vendor 의 실수 / Spoofing)
- Switch CAM table 의 entry 가 깜빡거림
- 진단: `arp -a` 결과의 변화

### 부분 충돌
다른 LAN 의 같은 MAC 은 문제 X (LAN 격리).

---

## 11. CAM Table / MAC Address Table

L2 Switch 가 학습한 MAC ↔ 포트 매핑.

```
+------------+----------+
| MAC        | Port     |
+------------+----------+
| 00:1A:2B...| Gi0/1    |
| 00:50:56...| Gi0/2    |
| ...        | ...      |
+------------+----------+
```

- 학습 (Learning): 출발 MAC 기록
- 노화 (Aging): 5 분 후 삭제
- Flooding: 없으면 Broadcast

### 11.1 CAM Overflow Attack (MAC Flooding)

- 무한 가짜 MAC 으로 CAM 채움 → Switch 가 hub 처럼 동작 → 도청 가능
- 방어: **Port Security** — 포트당 학습 MAC 수 제한

---

## 12. 면접 / 토픽

1. **MAC 주소 구조** — OUI / NIC-specific.
2. **Unicast / Multicast / Broadcast MAC** 구분.
3. **MAC 주소 vs IP 주소**.
4. **Locally Administered MAC** — VM / Wi-Fi 무작위화.
5. **MAC Spoofing 의 위험**.
6. **CAM Overflow / Port Security**.
7. **ARP 의 역할**.
8. **IPv4 mcast → MAC mcast 매핑**.

---

## 13. 함정

### 함정 1 — MAC 영구 가정
NIC 교체 / VM 이동 / 무작위화로 변함.

### 함정 2 — MAC 만으로 인증
스푸핑 쉬움. 802.1X 사용.

### 함정 3 — Broadcast Storm
무한 루프 시 모든 host 가 Broadcast 처리 → CPU 폭증. STP 필요.

### 함정 4 — Wi-Fi 의 MAC 무작위화로 차단
호텔 / 회사 Wi-Fi 가 MAC 기반 trust 면 매번 새 등록 필요.

### 함정 5 — 가상 MAC 충돌
VMware 가 같은 MAC pool 사용 — 큰 VM 농장에서 충돌 가능 (OUI 외 24 bit 도 부족할 수 있음).

---

## 14. 학습 자료

- IEEE Standard 802 — MAC Address Format
- IEEE OUI 데이터베이스 — https://standards.ieee.org/products-services/regauth/oui/
- **TCP/IP Illustrated Vol.1** — Stevens
- Wireshark — MAC OUI vendor 자동 표시

---

## 15. 관련

- [[ethernet-frame]] — 프레임 내 MAC 필드
- [[../layer-3-network/arp]] — IP ↔ MAC
- [[switching-bridging]] — CAM table
- [[layer-2-data-link]] — 상위
