---
title: "패널 기술 (LCD / OLED / Mini-LED / Micro-LED)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, display, lcd, ips, va, tn, oled, qd-oled, mini-led, micro-led, hdr, response-time, burn-in]
---

# 패널 기술

**[[display|↑ 디스플레이]]**

> 같은 픽셀 / 같은 해상도 / 같은 HDR 라벨이라도 패널 기술이 다르면 결과물이 완전히 다름. 응답 시간 / 대비 / 색 / 휘도 / 번인의 trade-off 한 화면.

## 1. LCD 의 기본 동작

```
       빛 (백라이트, LED)
         ▲
         │
    ┌────┴──────┐
    │ Diffuser  │  빛을 균일하게
    └────┬──────┘
         │
    ┌────┴──────┐
    │ Polarizer │  ↑ (수직 편광 통과)
    └────┬──────┘
         │
    ┌────┴──────┐
    │ TFT array │  각 픽셀의 전압 제어 (thin film transistor)
    └────┬──────┘
         │
    ┌────┴──────┐
    │ Liquid    │  ← 전압으로 액정 정렬 회전
    │ Crystal   │
    └────┬──────┘
         │
    ┌────┴──────┐
    │ Polarizer │  ↓ (수평 편광 통과)
    └────┬──────┘
         │
    Color Filter (R / G / B)
         │
    ─────▼─── 화면
```

### 동작 원리
- 백라이트 = 백색 LED 백판 (예전 CCFL).
- 액정 셀이 전압으로 정렬 → 편광 회전량 조절 → 빛 통과량 결정.
- color filter 가 RGB 분리.
- 한 pixel = R/G/B 3 sub-pixel.

### LCD 의 한계
- 백라이트가 항상 켜져 → "완전한 검정" 불가 (작은 빛 새어 나옴).
- 응답 시간 = 액정 회전 속도 (ms 단위).
- 시야각이 액정 종류 의존.

## 2. LCD 종류

### TN (Twisted Nematic)
- 가장 오래된 기술 (1970s).
- 액정이 90° twisted.
- 전압 인가 시 untwist.

**장점**:
- **응답 가장 빠름** (1 ms typical, 0.5 ms 까지).
- 가장 싸다.

**단점**:
- 시야각 좁음 (위/아래 색 반전).
- 색 평범, 대비 낮음 (1000:1 정도).
- HDR 못 함.

**용도**: 일부 240+ Hz 경쟁 게이밍 모니터.

### IPS (In-Plane Switching)
- LG / Panasonic 개발 (1996).
- 액정이 평면에서 회전 (perpendicular 아닌 horizontal).
- 시야각 ↑, 색 ↑.

**장점**:
- 시야각 178°.
- 색 정확도 ↑ (sRGB / Adobe RGB / DCI-P3 광역).
- "IPS glow" — 검정 화면이 검은색이 아니라 살짝 푸름.

**단점**:
- 응답 4-8 ms.
- 대비 1000:1 정도 (VA 보다 낮음).

**용도**: 워크스테이션 / 일반 게이밍 모니터 / 노트북.

### IPS 변종
- **AHVA (Advanced Hyper Viewing Angle, AUO)** — IPS-like.
- **PLS (Plane to Line Switching, Samsung)** — IPS clone.
- **Fast IPS** — 응답 1 ms 까지.
- **Nano-IPS (LG)** — 광색역 IPS.

### VA (Vertical Alignment)
- 액정이 수직 정렬에서 휘어짐 (전압 인가 시).
- Samsung / Sharp 가 주력.

**장점**:
- 대비 ↑ (3000:1 ~ 5000:1).
- 색 ↑ (DCI-P3 95%+).
- IPS glow 없음.

**단점**:
- 응답 약간 느림 (smearing — 어두운 장면 잔상).
- 시야각이 IPS 보다 약간 좁음.

**용도**: 일부 TV, 일부 게이밍 (Samsung Odyssey VA).

### Mini-LED (LCD + zone backlight)

기본 LCD 위에 백라이트가 작은 LED 수천~수만 zone 으로 분할.

```
   Backlight zones (수천 개 mini-LED)
   ┌─┬─┬─┬─┬─┬─┬─┐
   │A│B│C│D│E│F│G│   각 zone 의 LED ON/OFF/dim 독립
   ├─┼─┼─┼─┼─┼─┼─┤
   │ │ │ │ │ │ │ │
   └─┴─┴─┴─┴─┴─┴─┘
        ↓
       LCD layer (위와 동일)
        ↓
       Color filter
        ↓
       화면
```

**장점**:
- 어두운 zone 끄면 검정에 가까운 결과 → HDR 인증 1000-1400 nit.
- 풀-화면 휘도 OLED 보다 높음.

**단점**:
- **Blooming** — 작은 zone 안에서 밝은 픽셀 주변에 빛 새는 현상.
- 비쌈.

**용도**:
- iPad Pro M4.
- MacBook Pro 14/16 Liquid Retina XDR.
- 일부 HDR 게이밍 모니터 (Samsung Neo G9, ASUS ROG PG32UQ).

### Mini-LED zone 수
- Pro Display XDR: 576 zones.
- iPad Pro 11/13: 2596 zones.
- MacBook Pro 14: 8064 (Mini-LED).
- 더 많은 zone = 더 precise.
- → 차세대 = **Mini-LED → Dual-cell LCD (한 layer 가 LCD black mask)** 또는 → OLED 직접.

## 3. OLED — 자발광 유기

각 픽셀이 직접 빛 발산. 백라이트 없음.

```
   각 sub-pixel = OLED material
      ↓ 전압
   유기 분자가 빛 발산 (R / G / B 또는 W)
      ↓
   color filter (W-OLED 경우)
      ↓
   화면
```

### 장점
- **무한 대비** — 픽셀 끄면 진짜 0 nit.
- 응답 < 1 ms.
- 시야각 ~ 180°.
- 얇음 (백라이트 없으니).
- 곡면 가능 (flexible OLED).

### 단점
- **번인 (burn-in)** — 같은 정적 이미지 (UI / HUD) 장시간 노출 → 유기 분자 열화 → 해당 픽셀 영구 어두워짐.
- 최고 휘도 LCD 보다 낮음 (특히 풀-화면 흰색).
- 가격 ↑.
- 청색 sub-pixel 의 수명이 가장 짧음.

### OLED Sub-pixel 구조

#### RGB Stripe
- R, G, B 각 sub-pixel 균등.
- Samsung S 시리즈 외 일부.

#### PenTile (Samsung)
- RGBG 패턴 — 녹 sub-pixel 2 배.
- 같은 면적에 더 적은 sub-pixel 로 보임.
- 텍스트가 "fringing" 보일 수 있음.
- 갤럭시 S 시리즈, iPhone (LTPO OLED).

#### WOLED (LG)
- 흰 OLED + RGBW color filter.
- 4 sub-pixel (R/G/B + W).
- W sub-pixel 이 휘도 ↑ 제공.
- LG OLED TV / Pixel 8 외.

#### QD-OLED (Samsung Display)
- 청색 OLED + 양자점 (red/green converter).
- 색 영역 DCI-P3 99%+, Rec.2020 80%+.
- WOLED 보다 색 ↑, 일부 모델 휘도 ↑.
- Samsung S95C, Sony A95L.

### OLED 종류

| 종류 | 출처 | 특징 |
| --- | --- | --- |
| **AMOLED** | Samsung 외 | 일반 OLED. 각 sub-pixel 의 TFT 가 active matrix. |
| **POLED** | LG | 플라스틱 substrate (flexible). |
| **LTPO OLED** | Apple / Samsung | LTPO (Low-Temperature Polycrystalline Oxide) TFT. 1-120 Hz 가변 freq → 전력 ↓. ProMotion (iPhone 15 Pro). |
| **MLA (Micro Lens Array, LG)** | LG | OLED 픽셀 위 마이크로 렌즈 → 외부 광 efficiency ↑ → 밝기 ↑ 30%+. |
| **3-stack OLED (META 2.0)** | LG | RGB 적층 → 밝기 4× ↑ (시제품 2024). |

### OLED 번인 방지

- **Pixel shift** — 화면 내용 1-2 pixel 단위 자동 이동.
- **Logo dimming** — 정적 logo 자동 어둡게.
- **Screen saver** — idle 시 자동 off.
- **Subpixel rotation** — 각 sub-pixel 의 부하 균등화.
- **TPC (Temporal Peak Luminance Control)** — 풀-화면 흰색 출력 시 dim.

### OLED 수명
- 청색 sub-pixel의 50% luminance 감소 = ~10,000-50,000 시간.
- 정상 사용 (4-5 시간 / 일) = 5-10 년.
- HDR 풀 화면 흰색 게임 / TV 광고 등은 가속.

## 4. Micro-LED — 차세대

자발광 무기 LED (μm 단위).

### 장점
- OLED 의 번인 / 휘도 한계 모두 해결.
- 매우 긴 수명.
- 매우 높은 휘도 (5000+ nit).

### 현 단계
- TV 100"+ / 대형 사이니지.
- **Samsung The Wall** — 110" Micro-LED TV (~$200K).
- **Apple Vision Pro** — 매우 작은 OLED 그대로 사용 (Micro-OLED).
- 일반 모니터 / 노트북 도입은 2027+.

### 제조 난점
- micro-LED transfer (millions of LED 를 wafer 에서 panel 에) 의 yield.
- 대량 양산 비용.

## 5. 해상도 / 갱신율 / 픽셀 밀도

### 일반 해상도

| 해상도 | 픽셀 | 일반 용도 |
| --- | --- | --- |
| 1080p / FHD | 1920×1080 | 일반 |
| 1440p / QHD | 2560×1440 | 게이밍 |
| 4K / UHD | 3840×2160 | 작업 / 영상 |
| 5K | 5120×2880 | Apple Studio Display |
| 6K | 6016×3384 | Pro Display XDR |
| 8K | 7680×4320 | 시연 / 대형 사이니지 |

### 픽셀 밀도 (PPI)

| 디바이스 | PPI |
| --- | --- |
| 일반 27" 1440p 모니터 | 109 |
| 일반 27" 4K 모니터 | 163 |
| MacBook Pro 16" (3456×2234) | 254 |
| iPhone 15 Pro (2556×1179) | 460 |
| Apple Vision Pro micro-OLED | 3386 |

### Refresh Rate

| Hz | 용도 |
| --- | --- |
| 60 | 일반 |
| 75 / 90 / 120 | 모바일 / ProMotion (iPhone) |
| 144 / 165 | 게이밍 |
| 240 / 360 / 480 | 경쟁 게이밍 |
| 540 / 720 | 시제품 |

### VRR (Variable Refresh Rate)

자세히는 [[display]] hub. 요약:
- 패널이 GPU 출력 frame 도착 시에만 업데이트.
- screen tearing 사라짐.

## 6. HDR — 휘도 / 색 영역

### HDR 인증

| 인증 | 휘도 (nit) | 비고 |
| --- | --- | --- |
| DisplayHDR 400 | 400 | 의미 거의 없음 |
| DisplayHDR 500 | 500 | local dimming required |
| DisplayHDR 600 | 600 | true HDR 의 시작 |
| DisplayHDR 1000 | 1000 | premium |
| DisplayHDR 1400 | 1400 | top tier |
| DisplayHDR TrueBlack 400/500/600 | 400-600 | OLED 전용 (검정 무한) |

### HDR 표준
- **HDR10** — open. 10-bit, 1000-10000 nit. static metadata.
- **HDR10+** — dynamic metadata (장면 별). Samsung 주도.
- **Dolby Vision** — 12-bit + dynamic metadata. Disney+ / Apple TV.
- **HLG (Hybrid Log-Gamma)** — 방송. backward compatible w/ SDR.

### 색 영역

| 공간 | 면적 (xy plane) | 용도 |
| --- | --- | --- |
| **sRGB / Rec. 709** | 0.111 | 웹 / SDR 영상 |
| **DCI-P3** | 0.152 (sRGB 의 +37%) | 영화 / 모바일 OLED |
| **Adobe RGB** | 0.151 | 프린트 |
| **Rec. 2020** | 0.211 | 차세대 |
| **Rec. 2100** | 0.211 | HDR (Rec.2020 색 + HDR transfer function) |

현 디스플레이의 Rec.2020 cover = 70-85% (top tier).

### 휘도 측정 — 다양한 패턴

- **APL 1%** — 1% window 의 밝기. OLED 의 burst 휘도 (1000+ nit).
- **APL 10%** — HDR 보통 측정 패턴.
- **APL 100%** — 풀-화면 흰색. OLED 가 낮음 (200-400 nit), LCD 가 높음 (600+).

→ 같은 "1000 nit HDR" 모니터라도 풀-화면 1000 nit 못 내는 경우 흔함.

## 7. 응답 시간 / 잔상

### GtG (Gray-to-Gray)
- pixel 색 전환 시간.
- 마케팅 1 ms 는 best case (specific transition).
- 평균 / worst case 가 더 의미.

### MPRT (Moving Picture Response Time)
- 사람 눈에 보이는 잔상 시간.
- **Black Frame Insertion (BFI)** 로 줄임 (검은 frame 삽입 → CRT-like).
- BFI 의 단점: 휘도 50% ↓ + 깜빡임.

### Overshoot
- 응답 빠르게 하려고 driving voltage 과구동 → coronas / inverse ghosting.
- 모니터 OSD 의 "Overdrive" 설정.

### 마우스 트래일 / Smearing
- 어두운 픽셀 → 밝은 픽셀 전환이 가장 느림.
- VA 패널의 약점.

## 8. PWM (Pulse Width Modulation) — 깜빡임

### 백라이트 / OLED 의 밝기 조절
- **PWM dimming**: LED 를 빠르게 ON/OFF, ratio 로 밝기 조절.
- **DC dimming**: 전류 자체 조절.

### PWM 의 문제
- 100-500 Hz PWM 은 사람 인지 가능 → 두통 / 눈 피로.
- 1000+ Hz PWM = 거의 무해.
- OLED 의 저휘도 dimming 이 PWM 인 경우 많음 (iPhone 의 240 Hz PWM).

### Flicker-free 모니터
- DC dimming 또는 매우 높은 PWM (10+ kHz).

## 9. 픽셀 매트릭스

```
   [R] [G] [B] [R] [G] [B] ...        Row 1
   [R] [G] [B] [R] [G] [B] ...        Row 2
   ...

   각 픽셀 = R + G + B sub-pixel
```

### Stripe arrangement (LCD / OLED RGB)
- R, G, B 가 좌→우 순서.

### PenTile (Samsung OLED)
- RGBG. 녹 2 배.

### WOLED (LG)
- RGBW. 흰색 추가.

### Bayer pattern (camera sensor)
- RGGB. 디스플레이는 아니지만 픽셀 sensing 표준.

## 10. 시야각

| 종류 | 일반 시야각 (밝기 50%) |
| --- | --- |
| TN | 160° (위/아래 색 반전) |
| VA | 178° |
| IPS | 178° |
| OLED | ~180° |

## 11. 화면 anti-glare 처리

### Glossy
- 매끄러운 표면.
- 색이 더 vivid 하지만 반사 ↑.

### Matte (Anti-glare)
- 거친 coating 으로 빛 분산.
- 색 약간 ↓, 반사 ↓.

### Nano-texture (Apple)
- ProDisplay XDR / Studio Display 옵션.
- 매우 정교한 anti-glare.
- 비쌈.

## 12. 패널 vendor 의 구조

### Top tier
- **LG Display** — WOLED, IPS 외.
- **Samsung Display** — QD-OLED, AMOLED.
- **BOE** — IPS, OLED.
- **Sharp / Foxconn** — IPS, ZIPS.

### 모니터 brand vs panel vendor
- 같은 panel 을 여러 모니터 brand 가 사용.
- LG OLED 패널 → LG / Sony / Panasonic TV.
- 같은 panel 도 firmware / electronics 차이로 결과 다름.

## 13. eARC / CEC / HDMI features

- **eARC (enhanced Audio Return Channel)** — TV → AVR / soundbar 의 Dolby Atmos 전달.
- **CEC (Consumer Electronics Control)** — 한 remote 로 여러 기기.
- **ALLM (Auto Low Latency Mode)** — 게임 모드 자동 진입.
- **QFT (Quick Frame Transport)** — frame 전송 latency ↓.

## 14. 디스플레이 측정 / 진단

```bash
# Linux
xrandr                              # X11 monitor 정보
wlr-randr                           # Wayland (sway 등)
hwinfo --monitor                    # 자세히

# macOS
system_profiler SPDisplaysDataType

# 색 보정 도구 (전용)
# DisplayCAL — 색 calibration
# Calman — 전문가용
# 측정기: X-Rite i1 Display Pro, Datacolor Spyder, Calibrite Display Pro
```

## 15. 함정

1. **HDR 라벨만 보고 구매** — DisplayHDR 400 은 실 HDR 아님 (400 nit + dimming 없음).
2. **VRR 범위 (LFC)** — 48-144 Hz 패널은 48 미만 시 LFC 가 frame 2 배 출력. 안 되는 패널은 tearing.
3. **OLED 번인 (UI / HUD 정적 이미지)** — 작업 노트북은 OLED 비추 또는 dock / 메뉴바 자동 hide.
4. **케이블이 보틀넥** — HDMI 2.1 표시라도 48 Gbps 미지원 케이블 흔함. Ultra High Speed 인증 확인.
5. **DSC 미지원 GPU** — Gen 5 모니터 풀 스펙 못 받음.
6. **macOS HDR** — 일부 게이밍 모니터는 SDR/HDR 전환이 매끄럽지 않음.
7. **PWM dimming 의 눈 피로** — OLED 저휘도 PWM 가 두통 유발.
8. **풀-화면 휘도 OLED 의 한계** — APL 100% 가 200-400 nit. 풀-화면 흰 문서 작업 시 LCD 보다 어두움.
9. **TN 의 시야각** — 위·아래 각도에서 색 반전 (마케팅 안 함).
10. **VA 의 잔상 (smearing)** — 어두운 장면 게임에서 거슬림.
11. **Mini-LED blooming** — 작은 밝은 객체가 어두운 배경에서 헤일로.
12. **QD 패널의 cadmium 함량** — EU RoHS 규제. cadmium-free QD 가 표준.
13. **MLA (Micro Lens Array) 의 시야각 trade-off** — 정면 휘도 ↑ 대신 측면 휘도 ↓.

## 16. 관련

- [[display]]
- [[../bus-io/usb-thunderbolt]] — DP Alt-mode / TB 5
- [[../power-cooling/psu-vrm]] — 대형 디스플레이 PSU
- [[../soc/soc-anatomy]] — 디스플레이 컨트롤러
