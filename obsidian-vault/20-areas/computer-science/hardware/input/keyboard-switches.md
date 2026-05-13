---
title: "키보드 스위치"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, input, keyboard, switch, cherry-mx, topre, hall-effect, optical]
---

# 키보드 스위치

**[[input|↑ 입력 장치]]**

## 1. 종류

| 종류 | 원리 | 키감 | 수명 |
| --- | --- | --- | --- |
| **Membrane (rubber dome)** | 고무 돔 압축으로 contact | 푹신 | ~5 M |
| **Scissor / butterfly** | 고무 돔 + 가위 메커니즘 | 얕은 약간 단단 | ~10 M |
| **Mechanical (스프링 + contact)** | 스프링 + 금속 접점 | 세그먼트 다양 | 50–100 M |
| **Optical** | 빛 차단 감지 | 매우 빠른 응답 | 100 M |
| **Hall Effect (자기)** | 자기장 변화 측정 | 정전식 아날로그 입력 | 100 M+ |
| **Topre** | 정전 + 스프링 | 균일 큰 키압 | 30–50 M |

## 2. Mechanical — Cherry MX 와 동류

Cherry MX 가 표준. 각 색이 다른 액추에이션 / 키감 / 소리.

| 색 | 종류 | 액추에이션 (g) | 클릭 / 텍타일 / 리니어 |
| --- | --- | --- | --- |
| MX Red | Linear | 45 | 부드러움 |
| MX Black | Linear | 60 | 무거움 |
| MX Brown | Tactile | 45 | 약한 텍타일 bump |
| MX Blue | Clicky | 50 | 큰 텍타일 + clicky sound |
| MX Silent Red | Linear silent | 45 | 조용 |
| MX Speed Silver | Linear | 45 | 짧은 액추에이션 거리 (1.2 mm) |

### 액추에이션 거리 / 총 이동
- 표준 = 액추에이션 2 mm / total 4 mm.
- Speed = 액추에이션 1.2 mm.
- low-profile = total 3.2 mm.

### Cherry 외 호환·대안
- **Gateron** — Cherry clone, 가격 ↓ + 부드러움 ↑ 평가.
- **Kailh** — 다양한 변종 (Box, Speed, Choc low-profile).
- **TTC, Outemu, Akko** — 중국제, 가성비.

## 3. Optical 스위치

- 키가 빛 통로를 차단 → 광 센서가 감지.
- 디바운싱 없이 0.2 ms 응답.
- **Razer Optical**, **Wooting Lekker**.

## 4. Hall Effect

- 자기장 강도로 키 위치 측정 — 연속 값.
- **analog input** 가능 — 키를 얕게 누르면 분당 입력, 깊게 누르면 빠르게.
- 게이밍에서 WASD 의 가속 제어 (Wooting), 액추에이션 깊이 사용자 설정.
- 예: **Wooting 60HE / 80HE**, **Razer Huntsman v3 Analog**, **SteelSeries Apex Pro**.

## 5. Topre

- 정전식 + 원뿔 스프링 + 고무 돔.
- 키압이 균일 (스위치 색에 따라 30 / 45 / 55 g).
- 소리 부드럽고 키감 푹신.
- 예: **HHKB**, **Realforce**, **Niz Plum**.

## 6. 키캡 / Layout

- **OEM / Cherry / SA / DSA / XDA / KAT** profile — 키 높이·곡률.
- **PBT vs ABS** — PBT 가 마찰 적고 표면 코팅 안 닳음.
- **dye-sublimation vs double-shot** — 글자 인쇄 방식. 둘 다 안 닳음.
- **layout**: ANSI / ISO / JIS / HHKB. 60% / 65% / 75% / TKL / 100%.

## 7. 게이밍 키보드 추가 지표

- **N-key rollover (NKRO)** — 모든 키 동시 인식. 기본은 6KRO.
- **Polling rate** — 보통 1000 Hz. 일부 게이밍 8000 Hz.
- **Per-key RGB** — Razer / Logitech / Corsair OPENRGB.

## 8. 함정

1. **Cherry 색 비교만 보고 결정** — 같은 "MX Brown" 라도 retooled (2018+) 와 old 가 키감 다름.
2. **OPTICAL = HALL EFFECT 동일** — 다름. Optical 은 on/off, Hall 은 analog.
3. **저렴 mechanical 의 chattering** — debouncing 미흡으로 한 번 누름이 2 회 입력.
4. **사무실에서 clicky (MX Blue)** — 소음 민감.
5. **Topre 의 RGB 부재** — 정전식 구조상 백라이트가 어려움. 일부 모델만 가능.
6. **무선 키보드 latency** — Bluetooth ~15 ms, 2.4 GHz dongle ~1 ms. 게이밍은 후자.

## 9. 관련

- [[input]]
- [[../bus-io/usb-thunderbolt]] — USB-HID 프로토콜
