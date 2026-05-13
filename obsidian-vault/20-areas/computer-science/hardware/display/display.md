---
title: "디스플레이 (Display) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, display, lcd, oled, refresh, hdr, vrr]
---

# 디스플레이 — Hub

**[[../hardware|↑ hardware]]**

## 0. 한 줄

**픽셀을 빛으로 만드는 패널 + 신호를 받아 픽셀에 전달하는 컨트롤러 + 시간 동기 (refresh / VRR) + 색·휘도 (HDR).**

## 1. 패널 기술
→ [[panel-types|패널 기술]]

| 종류 | 원리 | 대비 | 응답 |
| --- | --- | --- | --- |
| TN LCD | 액정 twisted | 낮음 | 1 ms |
| IPS LCD | in-plane | 중 | 4–8 ms |
| VA LCD | vertical | 높음 | 4–8 ms |
| Mini-LED | LCD + zone 백라이트 | 높음 (HDR) | LCD 수준 |
| **OLED** | 자발광 유기 | 무한 | <1 ms |
| **QD-OLED** | OLED + 양자점 | 높음 (색) | <1 ms |
| **Micro-LED** | 자발광 무기 LED | 무한 | <1 ms |

## 2. 해상도 / 갱신율

| 해상도 | 픽셀 | 용도 |
| --- | --- | --- |
| 1080p / FHD | 1920×1080 | 일반 |
| 1440p / QHD | 2560×1440 | 게이밍 |
| 4K / UHD | 3840×2160 | 작업 / 영상 |
| 5K / 6K | Apple Studio Display / Pro Display XDR | 영상 편집 |
| 8K | 7680×4320 | 시연 / 디지털 사이니지 |

| Refresh rate (Hz) | 용도 |
| --- | --- |
| 60 | 일반 |
| 75/90/120 | 모바일 / 게이밍 |
| 144/165 | 게이밍 |
| 240/360/480 | 경쟁 게이밍 |

## 3. VRR / G-Sync / FreeSync

- 패널이 GPU 출력 frame 도착 시에만 업데이트.
- **screen tearing** 사라짐 + **stuttering** 감소.
- **VESA Adaptive-Sync** = 표준.
- **NVIDIA G-Sync** = 자체 모듈 (Ultimate) 또는 Adaptive-Sync 호환 (Compatible).
- **AMD FreeSync** = Adaptive-Sync 호환.

## 4. HDR (High Dynamic Range)

- **HDR10** — open. 최대 1000~10000 nit. 10-bit color.
- **HDR10+** — dynamic metadata (장면 별).
- **Dolby Vision** — 12-bit + dynamic metadata. Disney+ / Apple TV.

### 인증 (DisplayHDR)
- **DisplayHDR 400 / 500 / 600 / 1000 / 1400 / TrueBlack 400/500/600**.
- 1000 nit 이상 + local dimming = HDR 의 실효 시작.

### 색 영역
- **sRGB** — 웹 / 일반.
- **DCI-P3** — 영화 / 모바일 OLED.
- **Rec.2020** — 차세대 (현재 디스플레이는 70~85% 정도).
- **Adobe RGB** — 프린트.

## 5. 인터페이스

| 인터페이스 | 최신 | 속도 |
| --- | --- | --- |
| HDMI 2.1 | 2017 | 48 Gbps |
| HDMI 2.2 | 2025 | 96 Gbps (8K 120 / 12K) |
| DisplayPort 2.1 | 2022 | 80 Gbps (UHBR20) |
| DP over USB-C (Alt-mode) | | 80 Gbps |
| Thunderbolt 5 | 2023 | 80–120 Gbps |
| 무선 (WiGig / Wi-Fi 7) | | 제한적 |

## 6. DSC (Display Stream Compression)

- VESA 의 시각적 무손실 압축. 4K 240 / 8K 60 같은 고대역폭에 사용.
- 1.4 dB 이내 손실 (눈으로 구분 불가).
- 일부 모니터 / OS 가 DSC 미지원 → 해상도 / refresh 다운그레이드.

## 7. 함정

1. **HDR 라벨만 보고 구매** — DisplayHDR 400 은 실 HDR 아님 (400 nit + dimming 없음).
2. **VRR 범위 (LFC)** — 48–144 Hz 패널은 48 미만으로 떨어지면 LFC (Low Framerate Compensation) 가 frame 을 2 배 출력. 안 되는 패널은 tearing.
3. **OLED 번인 (burn-in)** — 같은 정적 이미지 (UI / HUD) 장시간 표시 시 화소 열화. Pixel shift / 자동 dimming 으로 완화.
4. **케이블이 보틀넥** — HDMI 2.1 라벨이라도 48 Gbps 미지원 케이블 (Ultra High Speed 인증 없음).
5. **DSC 미지원 GPU** — Gen 5 모니터 풀 스펙 못 받음.
6. **macOS HDR** — 일부 게이밍 모니터는 SDR/HDR 전환이 매끄럽지 않음.

## 8. 관련

- [[panel-types]]
- [[../bus-io/usb-thunderbolt]] — DP Alt-mode / TB5
- [[../soc/soc-anatomy]] — 디스플레이 컨트롤러
