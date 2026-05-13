---
title: "키보드 스위치"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, input, keyboard, switch, cherry-mx, topre, hall-effect, optical, mechanical, nkro, polling]
---

# 키보드 스위치

**[[input|↑ 입력 장치]]**

> 키 1 개의 동작이 어떻게 변환되는지 — 매일 수천 번 누르는 작은 부품의 차이가 손목 / 생산성 / 게임 성능에 모두 영향.

## 1. 스위치 종류 한눈에

| 종류 | 원리 | 키감 | 수명 (cycles) | 가격 |
| --- | --- | --- | --- | --- |
| **Membrane (rubber dome)** | 고무 돔 압축 contact | 푹신, 무거움 | ~5 M | 매우 싸다 |
| **Scissor / butterfly** | 고무 돔 + 가위 메커니즘 | 얕은 약간 단단 | ~10 M | 싸다 |
| **Mechanical (스프링 + contact)** | 스프링 + 금속 접점 | 다양 (각 색별) | 50-100 M | 중 |
| **Optical** | 빛 차단 감지 | 매우 빠른 응답 | 100 M+ | 중-비쌈 |
| **Hall Effect (자기)** | 자기장 변화 측정 | analog 입력 | 100 M+ | 비쌈 |
| **Topre (정전)** | 정전 + 스프링 + 고무 돔 | 균일 큰 키압 | 30-50 M | 매우 비쌈 |

## 2. Membrane / Rubber Dome — 가장 흔함

### 동작
- 키 누르면 고무 돔 압축.
- 돔 안 접점이 PCB 트레이스에 닿음.

### 특징
- 가장 싸다.
- 사무용 키보드의 90%.
- 키감 푹신, 액추에이션 거리 깊음 (4 mm).
- 노이즈 적음.

### 단점
- 수명 짧음 (5 M cycle).
- typing feel 미세 변화.
- 다중 키 동시 (NKRO) 지원 제한 (6-key rollover 정도).

## 3. Scissor — 노트북 표준

### 동작
- 고무 돔 + 가위 모양 stabilizer.
- 키 cap 이 평행하게 이동 (membrane 보다 정밀).

### 특징
- 얕음 (2-3 mm 액추에이션).
- 일정 typing feel.
- MacBook (Magic Keyboard), Dell XPS 등.

### Butterfly (Apple 2015-2019 — 실패)
- scissor 의 변형. 더 얇은 키캡.
- 먼지 / 부스러기에 매우 취약.
- 2019 Apple class action lawsuit.
- 2019+ Magic Keyboard 로 회귀 (scissor).

## 4. Mechanical — Cherry MX 와 동류

Cherry MX 가 산업 표준. 1980s 개발 후 patent 만료 (2014) 로 다양한 clone.

### Cherry MX 메커니즘

```
       Key cap
         │
   ┌─────┴─────┐
   │   Stem    │  ← 색 별 다른 stem 모양
   │   ┌─┴─┐   │
   │   │ S │   │  ← Spring
   │   └─┬─┘   │
   │   ┌─┴─┐   │
   │   │Cnt│   │  ← Contact (metal leaf)
   │   └─┬─┘   │
   └─────┬─────┘
         │
       PCB pin
```

- 키 누름 → stem 이 contact 를 압축.
- 일정 깊이 (액추에이션 point) 에서 contact 가 PCB pin 닿음 → 입력.
- 스프링이 키를 다시 위로.

### Cherry MX 색별 특성

| 색 | 종류 | 액추에이션 (g) | 키감 | 소리 |
| --- | --- | --- | --- | --- |
| **MX Red** | Linear | 45 | 부드러움 | 조용 |
| **MX Black** | Linear | 60 | 무거움 | 조용 |
| **MX Silent Red** | Linear silent | 45 | 부드러움 + dampener | 매우 조용 |
| **MX Speed Silver** | Linear | 45 | 짧은 거리 (1.2 mm) | 조용 |
| **MX Brown** | Tactile | 45 | 약한 텍타일 bump | 조용 |
| **MX Clear** | Tactile | 65 | 큰 텍타일 + 무거움 | 조용 |
| **MX Blue** | Clicky | 50 | 큰 텍타일 + click sound | 시끄러움 |
| **MX Green** | Clicky | 80 | Blue + 무거움 | 시끄러움 |
| **MX White** | Clicky | 80 | Green silent | 조용한 click |

### 액추에이션 거리 / 총 이동

| | 액추에이션 | total |
| --- | --- | --- |
| 표준 (Red / Brown / Blue) | 2 mm | 4 mm |
| Speed (Silver) | 1.2 mm | 3.4 mm |
| Low-profile | 1.2-1.5 mm | 3.2 mm |

### Cherry MX 외 호환·대안

| Brand | 특징 |
| --- | --- |
| **Gateron** | Cherry clone, 부드러움 ↑ 평가. 가격 ↓. |
| **Kailh** | 다양한 변종 (Box, Speed, Choc low-profile). |
| **TTC** | 중국. Akko / NJ80 등 채택. |
| **Outemu** | 중국. 매우 저렴. quality variance ↑. |
| **Akko** | TTC OEM. |
| **Razer** | Razer Green/Yellow (Kailh OEM). |
| **Logitech GX** | GX Blue/Brown/Red (Romer-G 후속). |
| **JWK / Durock** | 신생, custom 키보드 community 인기. |
| **Holy Panda** | Halo + Panda 의 mod, 강한 텍타일. |

### Cherry MX retooled (2018+)
- 같은 "Brown" 라벨이라도 retooled 와 old 가 키감 미세하게 다름.
- mold 갱신 후 사용감 ↑.

## 5. Mechanical — 다른 메커니즘

### Topre
- 정전식 + 원뿔 스프링 + 고무 돔.
- 키압 균일 (스위치 색에 따라 30 / 45 / 55 g).
- 소리 부드러움.
- 키감 푹신 + tactile.
- 예: HHKB, Realforce, Niz Plum.

### Buckling Spring (IBM Model M)
- 스프링이 buckling 하면 hammer 가 capacitive 감지.
- 1980-90s 표준.
- 큰 click 소리.
- 무겁고 두꺼움.
- 일부 enthusiast 가 여전히 사용.

### Alps
- 90s 미국 / 일본 표준.
- White Alps, Cream Alps, Salmon Alps 등 variant.
- 단종.
- 일부 vintage keyboard (Apple Extended II) 에 사용.

## 6. Optical 스위치

### 동작
```
   Key 위치       ←  키 cap
       │
   Stem 차단판  ←  키 누르면 빛 통로 차단
       │
   ┌───────┐
   │  LED  │  ←  광원
   └───┬───┘
       │
   ┌───┴───┐
   │ Sensor│  ←  광 감지
   └───────┘
```

- 키 누르면 stem 이 빛 통로 차단 → sensor 감지.
- 물리 contact 없음.

### 장점
- **debouncing 안 필요** — 0.2 ms 응답.
- 물 / 먼지 영향 적음.
- 수명 100 M+ cycle.

### 종류
- **Razer Optical** — Razer Huntsman series.
- **Wooting Lekker** — Wooting analog optical (Hall + optical hybrid).
- **Kailh PG155x** — Kailh.

## 7. Hall Effect — Analog 입력 가능

### 동작
```
       Key cap
         │
   ┌─────┴─────┐
   │  Magnet   │  ← 키 안 자석
   └─────┬─────┘
         │
   ┌─────┴─────┐
   │ Hall      │  ← Hall sensor 가 자기장 강도 측정
   │ Sensor    │
   └───────────┘
```

- 키 위치 = 자기장 강도 = analog 값.
- 0 ~ 4 mm 사이 임의 깊이 인식 가능.

### 장점 — Analog Input
- 키를 얕게 = 분당 입력, 깊게 = 빠르게 (예: WASD 의 walk vs run).
- 액추에이션 깊이 사용자 설정.
- Rapid Trigger — 어디서든 release 즉시 다시 입력.
- 자석 / sensor 무접촉 → 수명 매우 길음.

### 게임 / FPS 의 장점
- 미세 movement (자동차 게임 throttle).
- Counter-Strike 의 "Snap Tap" / Rapid Trigger.
- 2024+ 게이밍 시장 큰 변화.

### 일부 제품에서 cheating 으로 분류
- Counter-Strike 2 의 일부 functionality (Snap Tap) 가 2024 Valve 가 ban.
- 자동 W/S 동시 입력 처리 같은 기능.

### 대표 제품
- **Wooting 60HE / 80HE** — analog full.
- **Razer Huntsman v3 Analog** — 일부 키 only analog.
- **SteelSeries Apex Pro** — OmniPoint sensor.
- **Keychron Q1 HE** — HE.

## 8. 키캡 / Profile

### Profile (높이 / 곡률)

| Profile | 높이 | 곡률 |
| --- | --- | --- |
| **OEM** | 높음 (Cherry 표준) | 행별 곡률 |
| **Cherry** | OEM 보다 약간 낮음 | 행별 곡률 |
| **SA** | 매우 높음 | 큰 곡률 |
| **DSA** | 평평 (uniform) | 평평 |
| **XDA** | 평평 | 평평 |
| **KAT** | 중간 (uniform) | 약간 곡률 |
| **MT3** | 큰 곡률 + sculpting | |

### 재질

| 재질 | 특징 |
| --- | --- |
| **ABS** | 가장 흔함. 시간 지나면 표면 닳음 (shine). |
| **PBT** | 마찰 적음, 표면 안 닳음. legend (글자) 가 안 사라짐. |
| **POM** | 매우 매끄러움. |
| **PC** | 투명. RGB 조합. |

### 인쇄 방식

- **Pad printing** — 표면 인쇄. 닳음.
- **Laser etching** — 표면 절단. 부분적 영구.
- **Double-shot** — 두 plastic 색 결합. 영구.
- **Dye-sublimation** — 색을 plastic 안에 침투. PBT 만 가능. 영구 + 색 진함.

## 9. Layout

### 표준 layout

| Layout | 키 수 | 의미 |
| --- | --- | --- |
| Full / 100% | 104 (ANSI) / 105 (ISO) | numpad + nav 포함 |
| TKL / 80% | 87 / 88 | numpad 없음 |
| 75% | ~84 | nav 포함, F-row 좁음 |
| 65% | ~68 | F-row 없음, nav arrow |
| 60% | 61 | function-layer 만 nav |
| 40% | ~40 | 매우 작음, 모든 게 layer |
| **HHKB** | 60 | HHKB specific (Ctrl 자리 변경) |

### ANSI vs ISO

- **ANSI** = 미국. Enter 가로형.
- **ISO** = 유럽. Enter 가 L-자형. left shift 짧음.
- **JIS** = 일본. ISO 비슷 + 한자 key.

## 10. NKRO / Polling Rate

### N-Key Rollover (NKRO)
- 동시에 모든 키 인식.
- 기본 6KRO (USB HID limit).
- 6+ 키 동시 누름 시 일부 무시.
- **NKRO** = 모든 키 동시 보고.

### NKRO 구현
- HID over USB: USB Boot 모드는 6KRO, full HID 는 NKRO 가능.
- PS/2: native NKRO.
- HID over Bluetooth: 6KRO 일반적.

### Polling Rate
- 키보드가 USB report 보내는 빈도.
- 표준 125 Hz (8 ms).
- 게이밍 1000 Hz (1 ms).
- 최고 8000 Hz (0.125 ms).

### Polling rate 의 실 효과
- 게이밍 latency = polling + USB + OS scheduling.
- 1000 Hz vs 8000 Hz 의 인지 차이 거의 없음 (사람 reaction 200 ms).
- e-sports 에서 marginal.

## 11. 게이밍 키보드 추가 지표

### Per-key RGB
- 각 키 별 RGB LED.
- Razer Chroma / Corsair iCUE / Logitech G HUB / OpenRGB.
- 효과 / wallpaper sync.

### Macro keys
- 별도 macro 키 또는 layer.
- 일부 키보드의 dedicated G1-G6.

### Wireless
- **Bluetooth** — ~15 ms latency. 일반 작업 OK.
- **2.4 GHz dongle** — ~1 ms latency. 게이밍.
- **Wired USB** — 0 ms.

### Hot-swappable
- 솔더링 없이 스위치 교체.
- 다양한 스위치 시도 / 고장 시 교체.
- custom keyboard scene 의 표준.

## 12. Custom Keyboard 문화 (2018+)

### Group buy
- 작은 batch 키보드 / 키캡 / 스위치를 group order.
- 6-12 개월 대기.
- typical $300-$2000.

### Build 단계
1. PCB 선택.
2. Case (alu / acrylic / wood / brass).
3. Plate (steel / brass / FR4 / POM).
4. Stabilizer (Cherry / Durock V2 / Zeal).
5. Switch.
6. Keycap.
7. Foam / sound dampening.
8. Lubrication (스프링 / stem).

### 사운드 modding
- 스프링 lubrication.
- 스위치 안 stem lubrication.
- Case foam.
- Plate foam.
- PE foam (sounds-improving sheet).
- Tape mod (PCB 바닥에 마스킹).

## 13. QMK / VIA — Firmware

### QMK
- 오픈소스 ARM firmware.
- 매우 유연 — 모든 키 매핑 / layer / macro.
- C 로 작성, 컴파일 필요.

### VIA
- QMK 의 GUI 설정.
- 컴파일 없이 매핑 변경.

### ZMK
- BLE 기반. 무선 키보드.

### Vendor 의 자체 software
- Razer Synapse, Corsair iCUE, Logitech G HUB.
- Cloud 동기, RGB, macro.

## 14. 진단

```bash
# Linux 키 이벤트
sudo evtest                            # interactive

# Layout 확인
setxkbmap -query
localectl

# 실시간 입력
showkey -a                             # ASCII
showkey -k                             # keycode

# macOS
# System Settings > Keyboard
# Karabiner-Elements (custom mapping)

# Layout 변경
sudo dpkg-reconfigure keyboard-configuration
```

## 15. Ergonomic — RSI 방지

### 손목 stress
- 정면 자세, 손목 곧게.
- wrist rest 사용 (lifted 자세).
- 키압 너무 무거우면 손가락 stress ↑.

### Split / Ortholinear
- **Split keyboard** — 키보드를 두 부분으로 (Kinesis, Ergodox, Moonlander).
- **Ortholinear** — 키를 grid 정렬 (Planck, Preonic).
- staggered (일반) 의 손목 wrist twist 줄임.

### Tenting
- 키보드 가운데를 올려 손목 자연 자세.

## 16. 한국어 / 일본어 / 한자 입력

### 한국어
- 두벌식 / 세벌식 keyboard layout.
- Hangul/Han toggle key (보통 right alt).

### 일본어 (JIS)
- 변환 / 무변환 키.
- Romaji ↔ Kana 변환.

### 한자
- IME (Input Method Editor) 가 처리.
- macOS 입력기 / Windows IME / Linux ibus / fcitx.

## 17. 함정

1. **Cherry 색 비교만 보고 결정** — retooled 와 old 의 키감 차이.
2. **OPTICAL = HALL EFFECT 동일 오해** — Optical 은 on/off, Hall 은 analog.
3. **저렴 mechanical 의 chattering** — debouncing 미흡 → 한 번 누름이 2 회 입력.
4. **사무실에서 clicky (MX Blue)** — 소음 민감.
5. **Topre 의 RGB 부재** — 정전식 구조상 백라이트 어려움.
6. **무선 키보드 latency** — Bluetooth ~15 ms, 2.4 GHz dongle ~1 ms. 게이밍은 후자.
7. **NKRO 의 USB Boot 모드** — BIOS / 일부 OS 가 6KRO 만 인식.
8. **PBT 가 항상 좋다** — 일부 PBT 의 thin shot 이 cheap 느낌.
9. **Hot-swap 표시 = 모든 스위치 호환** — 일부 스위치 (Kailh BOX 등) 의 pin diameter 다름.
10. **Polling 8000 Hz 의 USB overhead** — 일부 OS / USB hub 가 폭주.
11. **Optical 스위치의 LED 수명** — LED 가 dim 해지면 응답 ↓.
12. **Group buy 의 환불 불가** — 6-12 개월 대기 + cancellation 어려움.

## 18. 관련

- [[input]]
- [[../bus-io/usb-thunderbolt]] — USB-HID 프로토콜
- [[../../security-theory/security-theory]] — keylogger HW vs SW
