---
title: "USB·Thunderbolt"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, usb, thunderbolt, usb-c, usb-pd, alt-mode, displayport, dma]
---

# USB·Thunderbolt

**[[bus-io|↑ 버스·I/O]]**

> 외부 장치의 표준 인터페이스. USB-C 커넥터에 USB / TB / DP / HDMI / PD 가 모두 모임 → 같은 모양 케이블이 다른 사양인 혼란의 근원.

## 1. USB 진화 — 표

| 표준 | 출시 | 속도 | 커넥터 | 비고 |
| --- | --- | --- | --- | --- |
| USB 1.0 | 1996 | 1.5 Mbps | A/B | low speed |
| USB 1.1 | 1998 | 12 Mbps | A/B | full speed. 마우스 / 키보드. |
| USB 2.0 | 2000 | 480 Mbps | A/B/Mini/Micro | high speed |
| USB 3.0 → 3.2 Gen 1 | 2008 | 5 Gbps | A/B/Micro 3.0 | SuperSpeed |
| USB 3.1 Gen 2 → 3.2 Gen 2 | 2013 | 10 Gbps | A/B/C | SuperSpeed+ |
| USB 3.2 Gen 2×2 | 2017 | 20 Gbps | C | 2 lane |
| **USB4** | 2019 | 20 / 40 Gbps | C | Thunderbolt 3 와 호환 |
| **USB4 v2** | 2023 | 80 Gbps 대칭 / 120+40 비대칭 | C | Thunderbolt 5 의 기반 |

→ USB-IF 의 명명이 여러 번 바뀌어 같은 케이블·포트라도 다른 이름. **케이블·디바이스 사양 라벨** 확인이 정답.

### USB 4 의 통합
- USB 3.2, DisplayPort, PCIe tunneling, USB-PD 까지 한 spec.
- Thunderbolt 3 사양 흡수.

## 2. USB-C 커넥터

```
   12345678   ← 위
   ─────────
   ─────────
   87654321   ← 아래 (대칭)

   24 pin total
```

### 핵심 핀
- A 1-12 / B 12-1 (대칭).
- 4 pin = power (Vbus / GND).
- 2 pair = USB 2.0 D+/D-.
- 4 pair = USB 3 SuperSpeed (TX1, RX1, TX2, RX2).
- 2 pin = CC1, CC2 (Configuration Channel, PD 통신).
- 2 pin = SBU (Sideband Use, Alt-mode).

### 양방향
- CC pin 으로 어느 방향 꽂혔는지 감지.
- 시작 시 PD chip 이 자동 negotiate.

### 케이블의 e-marker
- 5 A 이상 또는 10 Gbps 이상 케이블 = e-marker chip 필수.
- chip 이 자기 사양 (속도, 전력, vendor) PD 메시지로 보고.

## 3. USB Power Delivery (USB-PD)

### PD 세대

| 표준 | 최대 전력 | 비고 |
| --- | --- | --- |
| USB-PD 2.0 | 100 W (20 V × 5 A) | 노트북 충전 가능 |
| USB-PD 3.0 | 100 W + PPS | 가변 전압 (PPS) 빠른 충전 |
| **USB-PD 3.1 EPR** | **240 W** (48 V × 5 A) | 고출력 노트북 / 워크스테이션 |
| USB-PD 3.2 | 240 W + 개선 | |

### PPS (Programmable Power Supply)
- 충전 진행에 따라 V 조절 (20 mV step).
- 더 빠른 충전 + 효율.
- 갤럭시 super-fast / OPPO SuperVOOC 의 기반.

### EPR (Extended Power Range, PD 3.1)
- 28V / 36V / 48V 추가.
- 240 W 노트북 / 워크스테이션 충전.
- EPR 케이블 별도 (48V 5A 통과 인증).

### 동작
1. 케이블 끼움 → 양쪽 chip 이 CC 로 신호 교환.
2. SOURCE 가 Source Capability 메시지 (가능한 V/A 조합 list).
3. SINK 가 그중 하나 선택.
4. SOURCE 가 그 V/A 공급.
5. SINK 가 charging.
6. 변경 시 (느려질 때 등) 다시 negotiate.

### 케이블 정격 미스매치
- 5 W charger 에 240 W 케이블 = 5 W 동작 (낮은 쪽 fix).
- 240 W charger + 60 W 케이블 = 60 W (케이블 정격 한계).

## 4. Thunderbolt

| 세대 | 출시 | 속도 | 커넥터 | 비고 |
| --- | --- | --- | --- | --- |
| Thunderbolt 1 | 2011 | 10 Gbps | Mini DisplayPort | Apple + Intel |
| Thunderbolt 2 | 2013 | 20 Gbps | Mini DP | |
| **Thunderbolt 3** | 2015 | 40 Gbps | USB-C | PCIe x4 tunneling |
| **Thunderbolt 4** | 2020 | 40 Gbps | USB-C | 인증 강화. Dual 4K. PCIe 32 Gbps min |
| **Thunderbolt 5** | 2023 | 80 Gbps 대칭 / 120 Gbps 비대칭 | USB-C | USB4 v2 기반 |

### Thunderbolt 4 의 강제 요건
- Dual 4K display @ 60 Hz 또는 single 8K.
- PCIe 32 Gbps min (TB3 의 16 Gbps 에서 ↑).
- 모든 Thunderbolt 4 호스트가 USB4 호환.
- 4 port hub 가능.

### Thunderbolt 5 (Barlow Ridge)
- 80 Gbps 대칭.
- **bandwidth boost** = 120+40 Gbps 비대칭 (display 우선).
- Dual 6K @ 60 Hz 또는 Single 8K @ 60 Hz.
- 240 W charging (PD 3.1 EPR).
- 2024+ Apple M4 Pro/Max, Intel Arrow Lake / Lunar Lake.

## 5. PCIe Tunneling

Thunderbolt 의 핵심:

- PCIe lane 을 직렬화해서 케이블 위로 전송.
- 외부 GPU (eGPU), 외장 NVMe enclosure, Thunderbolt 도크 모두 PCIe 디바이스가 호스트에 직접 보임.
- IOMMU 활성 필수 — DMA 보호.

### 외부 NVMe enclosure
- TB3/4 = PCIe Gen3 x4 (16 Gbps) → 실 read ~2800 MB/s.
- TB5 = PCIe Gen4 x4 (32 Gbps) → 실 ~6 GB/s.
- 내부 NVMe (Gen4 x4 = 7 GB/s) 의 거의 같은 속도.

### eGPU
- Thunderbolt 가 외부 GPU 를 PCIe x4 로 연결.
- 게임 / 작업 시 성능 = 데스크탑 GPU 의 ~80%.
- TB5 = bottleneck 거의 사라짐.

## 6. Alt-mode (DisplayPort / HDMI)

USB-C 의 핀 일부를 DisplayPort / HDMI 신호로 재할당.

### DP Alt-mode
- USB 3.x 의 4 lane (2 TX + 2 RX) 중 일부 (또는 전부) 를 DisplayPort lane 으로.
- 4 lane 모두 DP = USB 2.0 만 가능.
- 2 lane DP + 2 lane USB 3 = 둘 다 가능.

### DisplayPort 2.1 over USB-C
- UHBR 13.5 / UHBR 20.
- 80 Gbps.
- 8K 60 / 4K 240.

### HDMI Alt-mode
- 거의 안 쓰임.
- DP Alt-mode + 도크 안 변환 chip 이 흔함.

## 7. 케이블 / 디바이스 호환성 매트릭스

| 케이블 등급 | 속도 | 충전 | 노트 |
| --- | --- | --- | --- |
| USB 2.0 케이블 | 480 Mbps 만 | 60 W (3 A) | 가장 흔한 USB-C 케이블 |
| USB 3.2 Gen 1 (5 G) | 5 Gbps | 60 W | 흔함 |
| USB 3.2 Gen 2 (10 G) | 10 Gbps | 60 W | |
| USB 3.2 Gen 2×2 (20 G) | 20 Gbps | 60 W | 드묾 |
| USB4 / TB3 passive (40 G, ≤0.8 m) | 40 Gbps | 100 W | 단거리만 |
| USB4 / TB3 active (40 G, 1-2 m) | 40 Gbps | 100 W | 케이블 안 칩 (active retimer) |
| **USB4 v2 / TB5 (80/120 G)** | 80/120 Gbps | 240 W | 1 m 까지, active |
| USB-PD EPR 케이블 | (속도 별도) | **240 W** | EPR 인증 |

→ **같은 모양에 케이블 등급 다름. 케이블이 보틀넥** 인 경우 흔함.

## 8. Active vs Passive 케이블

### Passive
- 단순 동선.
- 0.5-0.8 m 까지 (고속).
- 싸다.

### Active
- 케이블 내부에 retimer / amplifier chip.
- 1-2 m 까지 (40 G), 5 m 까지 (10 G).
- 비쌈.

### Optical Thunderbolt (Apple)
- 광 변환.
- 50+ m.
- 매우 비쌈.

## 9. USB 토폴로지 — Tree

USB 는 트리 구조 (hub).

### 호스트와 hub
- **Root hub** = 호스트 컨트롤러 (xHCI).
- **External hub** = 7 port hub 등.
- Tier 깊이 제한:
  - USB 2.0 = 7 tier.
  - USB 3.x = 7 tier.
  - USB4 / Thunderbolt = 6 hop max.

```bash
lsusb -t
# /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/8p, 480M
#     |__ Port 1: Dev 2, ...
```

### Bandwidth 공유
- 한 USB host controller 의 lane 을 그 tree 안 모든 device 가 공유.
- 4 USB 3.0 SSD 를 같은 hub 에 = 합 5 Gbps 까지만.

## 10. Thunderbolt Hub / Dock

### TB3/4 도크
- 1 TB 연결로 다중 장치:
  - 2-4 USB-A.
  - 1-2 USB-C (PCIe NVMe enclosure 등).
  - 1-2 DisplayPort / HDMI.
  - Ethernet.
  - SD 카드.
  - 60-100 W 노트북 충전 (passthrough).
- 동시 사용 시 BW 공유.

### TB5 도크
- 80 Gbps → 더 많은 동시 장치.
- Dual 4K @ 144 Hz 또는 single 8K.
- 240 W passthrough.

## 11. DMA Attack — Thunderbolt 의 위험

### Thunderspy (2020)
- Thunderbolt 의 PCIe tunneling 으로 외부 장치가 호스트 메모리 직접 read.
- 잠긴 컴퓨터에서도 일부 데이터 추출 가능.
- macOS / Linux / Windows 모두 영향.

### Thunderclap (2019)
- PCIe DMA 공격.
- IOMMU 우회 / 잘못 구성 시 위험.

### 대응
- **IOMMU 활성** 필수 (`intel_iommu=on iommu=pt` 또는 `amd_iommu=on iommu=pt`).
- Thunderbolt security level (BIOS):
  - **Security Level 0 (None)** — 모든 장치 자동.
  - **Level 1 (User Authentication)** — 사용자 확인 필요.
  - **Level 2 (Secure Boot)** — 서명된 장치만.
  - **Level 3 (DisplayPort only)** — 가장 강. PCIe tunneling 불가.

## 12. macOS 의 Thunderbolt

- macOS 가 Thunderbolt 의 native OS.
- 외부 GPU 지원 (Intel Mac 까지). Apple Silicon Mac 은 외부 GPU 미지원.
- Apple Silicon 의 Thunderbolt 4/5 가 USB4 와 동일 사양.

## 13. Linux 의 Thunderbolt

```bash
# Thunderbolt device 목록
sudo boltctl list

# 자동 authorize
sudo boltctl authorize ${UUID}

# Security level
cat /sys/bus/thunderbolt/devices/0-0/security
```

### thunderbolt-tools / bolt
- userspace daemon.
- security level / device authorization 관리.

## 14. eGPU 사용 — 실 사례

### 옛 (TB3, x4)
- RTX 4090 eGPU = PCIe Gen3 x4 (16 Gbps) bottleneck → 데스크탑의 ~60-70% 성능.
- 모니터 직출 시 80%+.

### TB5
- PCIe Gen4 x4 = 데스크탑 거의 동일 성능.
- macOS 는 Apple Silicon 에서 eGPU 안 됨 (driver 없음).

### Linux egpu-switcher
- 자동 GPU 선택.
- 작업 / 게임 동안 외장 GPU 사용.

## 15. 모바일 / 임베디드 USB

### USB OTG (On-The-Go)
- 모바일 디바이스가 host 역할.
- ID pin 으로 host/device 자동 결정.

### USB Gadget mode (Linux)
- 임베디드 Linux 가 USB device 흉내.
- Raspberry Pi Zero 가 USB 스토리지 / serial console 로 보이게.

## 16. 진단

```bash
# USB 트리
lsusb -tv
lsusb -d 1234:5678 -v          # vendor:product ID 자세히

# Thunderbolt
sudo boltctl list
sudo dmesg | grep -i 'thunderbolt\|tbt'

# USB-PD 협상 (USB-C device tree)
sudo dmesg | grep -i 'pd\|usb-c'

# 속도 확인
sudo lsusb -t                  # 480M / 5000M / 10000M 등 표시

# PCIe over Thunderbolt (eGPU 등)
sudo lspci | grep -i thunderbolt
sudo lspci -t                  # 트리에 TB 도크 + 그 아래 PCIe 장치 표시
```

## 17. 함정

1. **케이블이 보틀넥** — 같은 USB-C 커넥터에 480 Mbps 케이블이 들어가면 그 속도까지만. 케이블 라벨 / chip 확인.
2. **충전 안 됨** — PD 협상 실패 또는 케이블 PD 미지원. eMarker chip 있는 케이블 필요 (5 A 이상).
3. **eGPU 가 Thunderbolt 슬롯 인식 안 됨** — IOMMU / Thunderbolt security level 설정.
4. **외장 NVMe 발열** — 케이스가 발열 처리 못 하면 thermal throttling.
5. **USB-C 도크의 Alt-mode 충돌** — 모니터 + 외장 SSD + 노트북 충전 동시 시 lane 분배 불충분.
6. **USB 4 ≠ Thunderbolt 4 자동 호환** — USB4 v1 은 일부 TB3 기능이 옵션. 표시 확인.
7. **eMarker 없는 케이블에 240 W 시도** — 일부 케이블이 녹음. 정격 라벨 확인.
8. **DMA attack 가능성** — IOMMU 비활성 채 Thunderbolt 외부 장치 → 메모리 dump 위험.
9. **USB 도크에 너무 많은 장치** — 한 host controller bandwidth 공유. 4 SSD 가 5 Gbps 나눠씀.
10. **TB5 케이블 호환성** — 80 Gbps 표시 케이블 없으면 40 Gbps 만.
11. **Apple Silicon eGPU 시도** — driver 없어 동작 안 함 (2024 기준).
12. **USB-PD 의 D+ short 검출** — Apple charger 가 정품 인증 안 된 ESP 케이블에 충전 거부.

## 18. 관련

- [[bus-io]]
- [[pcie]] — Thunderbolt 가 터널링하는 layer
- [[dma-iommu]] — Thunderbolt 의 DMA 보호
- [[../display/display]] — DP Alt-mode, TB display
- [[../power-cooling/psu-vrm]] — USB-PD 의 EPR 240 W
