---
title: "USB·Thunderbolt"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, usb, thunderbolt, usb-c, usb-pd]
---

# USB·Thunderbolt

**[[bus-io|↑ 버스·I/O]]**

## 1. USB 진화

| 표준 | 출시 | 속도 | 비고 |
| --- | --- | --- | --- |
| USB 1.1 | 1998 | 1.5 / 12 Mbps | low / full speed |
| USB 2.0 | 2000 | 480 Mbps | high speed. 키보드 / 마우스 표준 |
| USB 3.0 (재명명: 3.2 Gen 1) | 2008 | 5 Gbps | SuperSpeed |
| USB 3.1 Gen 2 (재명명: 3.2 Gen 2) | 2013 | 10 Gbps | SuperSpeed+ |
| USB 3.2 Gen 2×2 | 2017 | 20 Gbps | 2 lane |
| **USB4** | 2019 | 20 / 40 Gbps | Thunderbolt 3 와 호환 |
| **USB4 v2** | 2023 | 80 Gbps 대칭 / 120+40 비대칭 | Thunderbolt 5 의 기반 |

> USB-IF 의 명명이 여러 번 바뀌어 같은 케이블·포트라도 다른 이름. **케이블·디바이스 사양 라벨** 확인이 정답.

## 2. USB-C 커넥터

- 양방향 (reversible).
- USB / Thunderbolt / DisplayPort / HDMI Alt-mode / USB-PD 모두 같은 모양.
- → **케이블이 같은 모양이라도 지원 프로토콜·속도·전력이 다름**.

## 3. USB Power Delivery (USB-PD)

| 표준 | 최대 전력 | 비고 |
| --- | --- | --- |
| USB-PD 2.0 | 100 W (20 V × 5 A) | 노트북 충전 가능 |
| USB-PD 3.0 PPS | 100 W + 가변 V | 빠른 충전 |
| USB-PD 3.1 EPR | **240 W** (48 V × 5 A) | 고출력 노트북 / 워크스테이션 |
| USB-PD 3.2 | 240 W + 개선 | |

### 동작
- 케이블의 e-marker 칩이 자기 정격 (5/20/60/100/240 W) 을 알림.
- 디바이스와 충전기가 PD 협상으로 V/A 결정.
- 5 W charger 에 240 W 케이블 끼우면 5 W 동작 (낮은 쪽 fix).

## 4. Thunderbolt

| 세대 | 출시 | 속도 | 비고 |
| --- | --- | --- | --- |
| Thunderbolt 1 | 2011 | 10 Gbps | mini DisplayPort 커넥터 |
| Thunderbolt 2 | 2013 | 20 Gbps | mini DisplayPort |
| Thunderbolt 3 | 2015 | 40 Gbps | USB-C 커넥터. PCIe x4 터널 |
| Thunderbolt 4 | 2020 | 40 Gbps | 인증 강화. Dual 4K. PCIe 32 Gbps min |
| **Thunderbolt 5** | 2023 | 80 Gbps 대칭 / 120 Gbps 비대칭 | USB4 v2 기반 |

Thunderbolt 4 / 5 = USB4 와 호환 + 강한 인증 (장치 보장, 더 많은 PCIe lane, 더 많은 디스플레이).

## 5. PCIe 터널링

Thunderbolt 의 핵심:
- PCIe lane 을 직렬화해서 케이블 위로 전송.
- 외부 GPU (eGPU), 외장 NVMe enclosure, Thunderbolt 도크 모두 PCIe 디바이스가 호스트에 직접 보임.
- IOMMU 활성 필수 — DMA 보호.

## 6. 케이블 / 디바이스 호환성 매트릭스

| 모양 | 케이블 등급 | 지원 속도 |
| --- | --- | --- |
| USB-C | USB 2.0 케이블 | 480 Mbps 만 |
| USB-C | USB 3.2 Gen 2 (10 Gbps) | 10 Gbps |
| USB-C | USB4 / TB3 passive (40 Gbps, ≤0.8 m) | 40 Gbps |
| USB-C | USB4 / TB3 active (40 Gbps, 1-2 m) | 40 Gbps |
| USB-C | USB4 v2 / TB5 (120 Gbps) | 120 Gbps |
| USB-C | USB-PD EPR | 240 W |

→ **같은 모양에 케이블 등급 다름. 케이블이 보틀넥** 인 경우 흔함.

## 7. Alt-mode (DisplayPort / HDMI)

USB-C 의 핀 일부를 DisplayPort / HDMI 신호로 재할당.

- **DP Alt-mode** — USB-C 한 케이블로 USB + DisplayPort.
- **HDMI Alt-mode** — 거의 안 쓰임.

DisplayPort 2.1 over USB-C = 80 Gbps. 8K 60 / 4K 240 가능.

## 8. USB 토폴로지

USB 는 트리 구조 (hub).

- **Root hub** = 호스트 컨트롤러.
- **External hub** = 7 포트 hub 등.
- Tier 깊이 제한 (USB 2.0 = 7 tier, USB4 = 다름).

```bash
lsusb -t
# /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/8p, 480M
#     |__ Port 1: Dev 2, ...
```

## 9. 함정

1. **케이블이 보틀넥** — 같은 USB-C 커넥터에 480 Mbps 케이블이 들어가면 그 속도까지만. 케이블 라벨 확인.
2. **충전 안 됨** — PD 협상 실패 / 케이블이 PD 미지원. eMarker 칩 있는 케이블 필요 (5 A 이상).
3. **eGPU 가 Thunderbolt 슬롯 인식 안 됨** — IOMMU / Thunderbolt security level 설정.
4. **외장 NVMe 발열** — 케이스가 발열 처리 못 하면 thermal throttling.
5. **USB-C 도크의 Alt-mode 충돌** — 모니터 + 외장 SSD + 노트북 충전을 동시에 했을 때 lane 분배가 불충분.
6. **USB 4 ≠ Thunderbolt 4 자동 호환** — USB4 v1 은 일부 TB3 기능이 옵션. 표시 확인.

## 10. 관련

- [[bus-io]]
- [[pcie]] — Thunderbolt 가 터널링하는 layer
- [[dma-iommu]] — Thunderbolt 의 DMA 보호
- [[../display/display]] — DP Alt-mode 의 디스플레이 면
