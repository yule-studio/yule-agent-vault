---
title: "패널 기술 (LCD / OLED / Mini-LED / Micro-LED)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, display, lcd, ips, va, tn, oled, qd-oled, mini-led, micro-led]
---

# 패널 기술

**[[display|↑ 디스플레이]]**

## 1. LCD 의 기본

```
       빛 (백라이트)
         ▲
         │
    ┌────┴────┐
    │ Diffuser │
    └────┬────┘
         │
    ┌────┴────┐
    │ Polarizer│  ↑
    ├────┬────┤
    │ Liquid Crystal │  ← 전압으로 정렬 조절
    ├────┬────┤
    │ Polarizer│  ↓
    └────┬────┘
         │
    Color Filter (R/G/B)
         │
    ─────▼─── 화면
```

- 백라이트 = 흰빛 (CCFL 옛, LED 현).
- 액정이 편광 각도 조절 → 빛 통과량.
- color filter 로 RGB 분리.

## 2. LCD 종류

### TN (Twisted Nematic)
- 가장 오래된 기술.
- 응답 가장 빠름 (1 ms).
- 시야각 좁음, 색 평범, 대비 낮음.
- 일부 240+ Hz 경쟁 게이밍 모니터에 잔존.

### IPS (In-Plane Switching)
- 액정이 평면에서 회전.
- 시야각 ↑, 색 ↑, 대비 중간.
- 응답 4-8 ms.
- 워크스테이션 / 일반 게이밍 모니터의 표준.

### VA (Vertical Alignment)
- 액정이 수직 정렬에서 휘어짐.
- 대비 ↑ (3000:1+), 색 ↑.
- 응답 약간 느림 (smearing).
- TV / 일부 게이밍.

### Mini-LED (LCD + zone backlight)
- 백라이트가 작은 LED 수천~수만 개의 zone 으로 분할.
- 어두운 zone 끄면 검은색에 가까운 결과.
- HDR 인증 1000~1400 nit.
- 단점: 작은 zone 안에서 blooming (밝은 픽셀 주변 빛 새는 현상).
- 예: iPad Pro M4, MacBook Pro 14/16 Liquid Retina XDR.

## 3. OLED — 자발광 유기

각 픽셀이 직접 빛 발산. 백라이트 없음.

- **무한 대비** — 픽셀 끄면 진짜 0 nit.
- 응답 <1 ms.
- 시야각 매우 ↑.
- 단점:
  - **번인 (burn-in)** — 같은 정적 이미지 장시간 노출 → 화소 열화.
  - 최고 휘도 LCD 보다 낮을 수 있음 (특히 풀-화면 흰색).
  - 가격 ↑.

### OLED sub-pixel 구조
- **RGB stripe** — 균등 픽셀.
- **PenTile (Samsung 일부)** — RGBG. 녹 sub-pixel 2 배. 텍스트가 fringing 보일 수 있음.
- **WOLED (LG)** — 흰 OLED + color filter.

## 4. QD-OLED (Quantum Dot OLED)

- Samsung Display 2022. OLED 청색 emitter + 양자점 (red/green converter).
- 색 영역 DCI-P3 99%+ / Rec.2020 80%+.
- WOLED 보다 색 ↑, 일부 모델 휘도 ↑.
- 예: Samsung S95C, Sony A95L.

## 5. Micro-LED

- 자발광 무기 LED (μm 단위).
- OLED 의 번인 / 휘도 한계 모두 해결.
- 현 단계: TV 100"+ / 대형 사이니지 (Samsung The Wall).
- 가격이 매우 높아 일반 모니터 / 노트북 도입은 2027~.

## 6. 응답 시간 / 잔상

- **GtG (Gray-to-Gray)** — pixel 색 전환 시간. 마케팅 1 ms 는 best case.
- **MPRT (Moving Picture Response Time)** — 사람 눈에 보이는 잔상 시간. **black frame insertion (BFI)** 으로 줄임.
- **overshoot** — 응답을 빠르게 하려고 과구동 → coronas 잔상.

## 7. 시야각

| 종류 | 일반 시야각 (밝기 50%) |
| --- | --- |
| TN | 160° |
| VA | 178° |
| IPS | 178° |
| OLED | ~180° |

## 8. 함정

1. **TN 의 시야각** — 위·아래 각도에서 색 반전.
2. **VA 의 잔상 (smearing)** — 어두운 장면 게임에서 거슬림.
3. **Mini-LED blooming** — 작은 밝은 객체 (커서 / HUD) 가 어두운 배경에서 헤일로.
4. **OLED 번인 (특히 작업용 노트북)** — 항상 켜진 dock / 메뉴바 / Slack 채널 목록 부담.
5. **HDR 휘도 측정** — 풀-화면 흰색 (FullField) 휘도 vs 10% window 휘도 가 다름. OLED 는 10% window 만 높고 풀-화면은 낮음.

## 9. 관련

- [[display]]
- [[../power-cooling/psu-vrm]] — 대형 디스플레이 PSU
