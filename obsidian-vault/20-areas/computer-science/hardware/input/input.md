---
title: "입력 장치 (Input Devices) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, input, keyboard, mouse, touch, stylus]
---

# 입력 장치 — Hub

**[[../hardware|↑ hardware]]**

## 0. 한 줄

**사람의 의도를 디지털 신호로 변환하는 장치 — 키보드 / 마우스 / 터치 / 펜 / 음성 / 카메라 / 센서.**

## 1. 키보드
→ [[keyboard-switches|키보드 스위치]]

Membrane / Mechanical (Cherry MX 등) / Optical / Hall Effect / Topre.

## 2. 마우스 / 트랙패드

### 센서
- **광학 (LED)** — 표면 사진을 고속 촬영해 차분으로 이동량 계산.
- **레이저** — 더 다양한 표면. 일부 게이밍에서 jitter 이슈로 광학 선호.

### 지표
- **DPI** (사실은 CPI, Counts Per Inch) — inch 당 카운트. 800–3200 일반, 게이밍 25000+.
- **Polling rate** — USB report 빈도. 125 / 500 / 1000 / 8000 Hz.
- **Lift-off distance** — 들었을 때 동작 멈추는 거리.
- **Click latency** — 클릭 → OS 도달 시간 (ms). 일부 게이밍은 0.5–1 ms.

### 트랙패드
- Apple Force Touch — Taptic Engine 진동으로 클릭 시뮬레이션.
- Windows Precision Touchpad — 표준 드라이버.

## 3. 터치

### 정전 용량식 (Capacitive)
- 표준 스마트폰.
- **Self-cap** — 단일 손가락. 한 축씩.
- **Mutual-cap** — 격자 교차점 정전 용량 측정. 멀티 터치.

### 저항식 (Resistive)
- 두 막이 압력으로 접촉. 옛 PDA / 일부 산업 단말기.
- 손톱 / 장갑도 동작 (정전식 대비 장점).

### 호버 / 압력
- Samsung S Pen 호버 (정전 + 자기).
- Apple Force Touch (3D Touch) — iPhone 6s~11.

## 4. 펜 / 디지타이저

| 기술 | 원리 | 예 |
| --- | --- | --- |
| **Wacom EMR** | 전자기 유도 — 펜 배터리 불필요 | 갤럭시 노트, Wacom 태블릿 |
| **Apple Pencil 2/Pro** | Bluetooth + 정전 | iPad |
| **Microsoft Pen Protocol (MPP)** | 정전 + 압력 + tilt | Surface |
| **USI (Universal Stylus Initiative)** | 표준 정전 | Chromebook 등 |

압력 단계: 256 → 1024 → 4096 → 8192 → 16384.

## 5. 음성 / 카메라 / 센서

- **마이크 어레이** — 4-8 마이크 + DSP 로 빔포밍.
- **얼굴 인식 (Face ID)** — IR dot projector + IR camera + depth sensor.
- **지문 인식** — 정전식 (Touch ID) / 광학 / 초음파.
- **가속도계 / 자이로 / 자력계** — sensor fusion 으로 자세 / 흔들림 감지.
- **UWB (Ultra-Wideband)** — Apple U1/U2 칩. 거리·방향 측정 cm 단위.

## 6. 함정

1. **DPI 만 보고 게이밍 마우스 선택** — sensor 의 jitter / lift-off / 정확도가 더 중요.
2. **저가 mechanical 키보드의 switch 변종** — Cherry 가 아닌 clone 일 수 있음. 같은 색이라도 다른 액추에이션 거리.
3. **터치스크린의 false palm rejection** — 펜 사용 시 손바닥 무시 알고리즘. OS 의존성.
4. **USB polling 1000 Hz 라도 OS / 응용에서 60 Hz 로 처리** — 응용이 RAF 동기 시 사실상 무용.
5. **Bluetooth 키보드의 latency / 끊김** — 같은 2.4 GHz 대역의 Wi-Fi / 다른 BT 와 간섭.

## 7. 관련

- [[keyboard-switches]]
- [[../display/display]] — 터치 + 화면 결합
