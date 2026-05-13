---
title: "PSU 와 VRM"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, power, psu, vrm, atx, 80plus, 12vhpwr, drmos, transient, pmic, ocp, pfc]
---

# PSU 와 VRM

**[[power-cooling|↑ 전원·쿨링]]**

> 전기가 콘센트에서 트랜지스터의 게이트까지 도달하는 모든 변환 단. PSU 한 곳 fail = 시스템 전체 정지.

## 1. PSU — 콘센트에서 시스템까지

### 전체 변환 흐름

```
   AC 콘센트 (110V / 220V, 50/60Hz)
        │
   ┌────▼────┐
   │   PFC   │  Power Factor Correction (Active)
   └────┬────┘
        │
   ┌────▼────┐
   │  Diode  │  AC → DC (rectification)
   └────┬────┘
        │ ~400V DC
   ┌────▼────┐
   │ Primary │  switching (PWM, ~100 kHz)
   └────┬────┘
        │
   ┌────▼────┐
   │  Trans  │  Isolation transformer (galvanic)
   └────┬────┘
        │
   ┌────▼─────────────┐
   │ Secondary +      │  12V / 5V / 3.3V / -12V / 5Vsb
   │ rectification    │
   └────┬─────────────┘
        │
   ┌────▼────┐
   │ Filter  │  capacitor + inductor
   └────┬────┘
        │
       ATX 24-pin / EPS 8-pin / PCIe 8-pin / 12V-2x6 등
```

### 출력 rail

| Rail | 전압 | 용도 |
| --- | --- | --- |
| +12V | 12 V | CPU, GPU, fans, motors |
| +5V | 5 V | USB, drives, low-level chips |
| +3.3V | 3.3 V | RAM, chipset |
| -12V | -12 V | legacy (RS-232) — 현 거의 미사용 |
| +5Vsb | 5 V standby | 시스템 꺼져도 활성 (WoL, RTC) |

→ 현 PSU 의 99% 가 12V rail.

## 2. ATX 표준 폼팩터

### ATX (Standard)
- 150 × 86 × 140 mm.
- 데스크탑 풀사이즈 케이스.
- 가장 흔함.

### SFX (Small Form Factor)
- 100 × 63.5 × 125 mm.
- mITX / SFF 케이스.
- 일부 800W+ 모델.

### SFX-L
- SFX + 길이 ↑ = 더 큰 fan / 더 높은 W.

### TFX (Thin Form Factor)
- 슬림 / OEM 데스크탑.

### CRPS (Common Redundant Power Supply)
- 서버 hot-swap.
- 동일 chassis 의 PSU 2-3 개.
- N+1 redundancy.

### 데이터센터 (OCP)
- Open Rack 의 48V power shelf.
- 각 sled 가 48V 받아 자체 12V 변환.
- AI rack (GB200 NVL72) 의 표준.

## 3. 80 PLUS 인증 (효율)

| 등급 | 20% 부하 | 50% 부하 | 100% 부하 |
| --- | --- | --- | --- |
| 80 PLUS | 80% | 80% | 80% |
| Bronze | 82% | 85% | 82% |
| Silver | 85% | 88% | 85% |
| Gold | 87% | 90% | 87% |
| Platinum | 89% | 92% | 89% |
| Titanium | 90% | 94% | 90% (10% 부하 92%) |

### 효율의 의미
- PSU 효율 90% = 1000 W 출력하려면 1111 W 입력 (111 W 가 열로).
- 80% = 1250 W 입력 (250 W 열).
- 데이터센터 / 서버 = 90%+ 필수 (총 전력 / 열 부담).

### 80 PLUS 의 한계
- 인증이 110V 기준 (미국).
- 220V (한국 / 유럽) 는 같은 model 이 더 효율 (5%+).
- Cybenetics 인증이 더 정확 (3 voltage 측정).

## 4. PFC (Power Factor Correction)

### Power Factor 의 의미
- AC 의 전압 / 전류 phase 차이.
- 0 ~ 1. 1 이 ideal.
- 낮은 PF = 같은 W 출력에 더 큰 전류 필요 → 인프라 부담.

### Active PFC vs Passive PFC
- **Active PFC** = boost converter 가 입력 전류를 sinusoidal 로 만듦. PF ~0.99.
- **Passive PFC** = LC filter. PF ~0.6.
- 현 PSU 의 거의 모두 = Active PFC.

### 데이터센터 요구
- 0.99+ 필수.
- 그렇지 않으면 전기 회사 패널티.

## 5. PSU 모듈러

| 종류 | 의미 | 장단점 |
| --- | --- | --- |
| **Non-modular** | 모든 케이블 PSU 와 일체 | 싸다. 케이블 정리 어려움. |
| **Semi-modular** | 메인 24-pin / CPU EPS 만 고정, 나머지 분리 | 균형. |
| **Full-modular** | 다 분리 | 깔끔. 약간 비쌈. |

### 다른 PSU 의 modular 케이블 혼용 — 절대 금지
- 같은 모양 커넥터지만 핀 매핑 다름.
- 잘못 연결 시 12V 가 GND 에 → PSU / 메인보드 / 카드 burn.
- **반드시 같은 PSU 의 케이블** 사용.

## 6. 12V-2x6 (구 12VHPWR) — RTX 4090 / 5090

### 사양
- PCIe Gen5 부속 표준.
- 단일 커넥터 **600 W**.
- 12 main pin (6+6, 4-pair × 12V/GND) + 4 sense pin.
- sense pin 으로 케이블 / PSU 능력 협상.

### 12VHPWR vs 12V-2x6
- **12VHPWR** = original (2022).
- **12V-2x6** = revised (2024).
- 차이: sense pin 위치 / 깊이 개선. 부분 체결 시 fail safe.

### 녹는 사고 (2022-2024)
- RTX 4090 출시 후 다수 보고.
- 원인 분석:
  - 부분 체결 (끝까지 안 꽂힘) → 핀 접촉 면적 부족 → 발열 폭주.
  - 케이블 곡선이 커넥터 직후 너무 가파름 → 핀에 측면 압력.
  - 일부 어댑터 케이블의 quality 문제.

### 대응
- **끝까지 딸각** 들리도록 체결.
- 케이블이 커넥터 직후 ≥ 3 cm 직선 유지.
- 12V-2x6 표시 PSU + 케이블 사용.
- 정품 native cable 권장 (3rd-party 어댑터 비추).

## 7. PSU 사이징

### 공식

```
필요 W = (CPU PL2 전력) + (GPU TGP × 1.2) + (시스템 100~150 W)
권장 W = 필요 W × 1.3 ~ 1.5 (transient + 노화 margin)
```

### 예
- i9-14900K PL2 253 W + RTX 4090 450 W × 1.2 + 시스템 150 W = ~943 W.
- 권장 = 1200-1400 W 80+ Gold.

### Transient spike
- GPU 의 1-10 ms spike 가 정격의 **2 배** 까지.
- RTX 4090 = 600 W spike 가능 (TGP 450 W).
- PSU 의 transient response 가 부족하면 OCP 발동 → 시스템 셧다운.
- 게이밍 GPU 는 항상 OCP margin 큰 PSU 추천.

### 효율 sweet spot
- 50% 부하에서 PSU 효율 최고.
- 1000 W 시스템 → 1500 W PSU 가 효율 + 안정 ↑.
- 단, 너무 큰 PSU = idle 효율 ↓.

## 8. PSU 의 보호 회로

| 약자 | 의미 |
| --- | --- |
| **OCP (Over Current Protection)** | rail 별 전류 한계 초과 → 차단 |
| **OVP (Over Voltage)** | 12V 가 13V 초과 등 |
| **UVP (Under Voltage)** | load 폭증으로 전압 sag |
| **OPP (Over Power)** | 총 W 한계 |
| **OTP (Over Temperature)** | 내부 thermistor |
| **SCP (Short Circuit)** | rail 간 short |
| **NLO (No Load Operation)** | load 없이 켜기 보호 |
| **SIP (Surge & Inrush)** | 켜는 순간 spike 보호 |

### Single-rail vs Multi-rail 12V
- **Single-rail** = 모든 12V 가 한 OCP 회로 (큰 한계).
- **Multi-rail** = CPU / GPU / drive 별 별도 OCP.
- Multi-rail 의 안전 ↑, 하지만 단일 큰 부하 (GPU) 가 한 rail 한계 초과 가능.

### 데이터센터 PSU 의 redundancy
- N+1 = 1 PSU fail OK.
- 2N = 모든 PSU 가 2 개 (전체 capacity 가능).
- hot-swap 시 power good signal 으로 인계.

## 9. VRM (Voltage Regulator Module) — 메인보드 위

CPU 가 12 V 를 직접 못 받음 → 메인보드 VRM 이 1.0–1.4 V 로 변환.

### Multi-phase 토폴로지

```
12V input
     │
  ┌──┴──┐
  │     │
 phase1 phase2 ... phaseN
  │     │           │
  └──┬──┴───────────┘
     │
   Vcore (1.0-1.4V to CPU)
```

- 각 phase = MOSFET (High-side + Low-side) + inductor + capacitor.
- N 개 phase 가 시간 분산해서 전류 공급.
- 같은 freq 라도 N phase = ripple 1/N + 발열 1/N.

### Phase 의 효과
- 데스크탑 8-phase (X670E 보드) 가 250 W PL2 처리.
- 데스크탑 24-phase = 400 W spike 처리.
- 서버 EPYC = 30+ phase = 500 W TDP.

### Component

| 부품 | 역할 |
| --- | --- |
| **High-side MOSFET** | 12V → switch node |
| **Low-side MOSFET** | switch node → GND |
| **Driver** | gate 신호 생성 (gate drive IC) |
| **DrMOS / SmartPower Stage** | High + Low + Driver 한 칩 |
| **Inductor** | switching → DC |
| **Capacitor** | smoothing |
| **PMIC / Controller** | phase 사이 조율, current sense, OCP |

### DrMOS / SmartPower Stage
- Texas Instruments NexFET, Vishay SiC60xx, Infineon TDA21472 등.
- 한 칩이 70-110 A 처리.
- 발열 ↑ → heatsink 필요.

## 10. CPU 의 voltage rail

### 옛 (Intel Core 6th gen 까지)
- CPU 가 외부 VRM 으로부터 직접 Vcore 받음.

### Intel FIVR (Fully Integrated Voltage Regulator)
- Intel 4th gen (Haswell) ~ 5th gen.
- CPU 안에 voltage regulator 통합.
- 외부 VRM 은 IVR 에 단일 전압만 공급.
- 발열 ↑ 문제로 6th gen 부터 제거.

### 현 (Intel 12th-15th gen / AMD Ryzen 7000+)
- 외부 VRM 이 직접 Vcore / Vsoc / VDDR / VDDP 공급.
- 다수 voltage rail.

### Apple Silicon
- 자체 PMIC 가 통합.
- 외부 의존도 낮음.

## 11. RAM 의 voltage — DDR5 의 PMIC

### 옛 DDR4
- 메인보드 VRM 이 1.2 V 공급.

### DDR5
- **모듈 위 PMIC** 가 1.1 V 자체 생성.
- VRM 노이즈 격리 + 모듈 별 정밀 조절.
- OC 시 PMIC 의 phase / current limit 가 천장.

### Locked PMIC vs Unlocked PMIC
- 일부 PMIC (예: Renesas R29xx) 가 OC 제한.
- Unlocked PMIC = 1.4 V+ 가능.
- OC 전용 키트 (G.SKILL Trident Z5, Corsair Dominator Titanium) 는 unlocked 선택.

## 12. 전원 분배 — 데이터센터

### 12V (Open Compute Project Rack)
- 옛 → 효율 한계.

### 48V Rack Power
- Google / Meta / Microsoft 의 표준.
- AC → 48V DC rack-level.
- 각 sled / server 가 48V 받아 자체 12V (또는 직접 1V) 변환.
- 효율 ↑, 동선 ↓.

### 380V / 415V DC
- 가장 효율 (AC 변환 1 회 줄임).
- 일부 hyperscale 데이터센터.

### High-power AI rack (GB200 NVL72)
- 120 kW 한 rack.
- 48V power shelf 다수.
- liquid cooling 필수.

## 13. PSU 효율 곡선

```
Efficiency vs Load
   ▲
   │      ╱─────╲
92%│    ╱        ╲
   │   ╱          ╲
90%│  ╱            ╲
   │ ╱              ╲
85%│╱                ╲
   │                  ╲
   ├─────────────────────►
   0    25  50  75  100  Load %
```

- 20-80% 부하가 sweet spot.
- < 10% 또는 > 90% 는 효율 ↓.
- 그래서 1000 W 시스템에 1500 W PSU = 50-66% 부하로 sweet spot.

## 14. Inrush Current — 켜는 순간

- PSU 의 capacitor 가 비어 있어서 켜는 순간 큰 전류.
- typical inrush = 30-50 A peak (몇 μs).
- 다수 PSU 가 동시 켜지면 breaker trip.
- 데이터센터 = staggered startup.

## 15. PSU 의 노화

### Capacitor aging
- Electrolytic capacitor 가 시간 / 온도에 따라 capacitance ↓.
- 5-10 년 후 ripple 폭증.
- 데이터센터 = 7-10 년에 PSU 교체.

### Fan bearing wear
- PSU fan 의 ball bearing 마모.
- 시끄러워지면 곧 fan stop → PSU OTP 발동.

### 모니터링
- PSU 의 12V / 5V / 3.3V 가 정격 ±5% 안인지.
- ripple / noise 가 ±120 mV (ATX) 안인지.

## 16. 진단

```bash
# Linux 시스템 전원 / VRM 온도
sensors
sudo apt install lm-sensors
sudo sensors-detect

# Power capping (DCMI / OCM)
ipmitool dcmi power reading
ipmitool dcmi power get_limit

# Intel / AMD power 모니터링
sudo apt install powertop
sudo powertop

sudo apt install s-tui
s-tui                                  # interactive

# Apple Silicon
sudo powermetrics --samplers cpu_power,gpu_power -i 1000

# 직접 PSU 측정 (필요 시)
# - kill-a-wat 가정용 power meter.
# - 서버 PSM (Power Supply Monitoring) IPMI.
```

## 17. UPS (Uninterruptible Power Supply)

### Online (Double Conversion)
- 항상 배터리 → 인버터 → 시스템.
- 입력 AC 끊기면 0 ms 전환.
- 데이터센터 / 미션 크리티컬.

### Line-interactive
- 평상시 AC 직접 + 배터리 충전.
- AC 끊기면 ~10 ms 전환.
- SMB 표준.

### Standby (Offline)
- 평상시 AC 직접, AC 끊기면 ~20 ms 배터리 전환.
- 가정용.

### Capacity
- VA (kVA) vs W (kW).
- 0.7-0.9 power factor → VA × 0.7 ≈ 실 W 한계.

## 18. 함정

1. **PSU 정격 = 시스템 W** 일 때 transient spike 로 셧다운 — 마진 30%+ 두기.
2. **12V-2x6 부분 체결** — 녹는 사고. 끝까지 체결 + 곡선 완만.
3. **저가 PSU 의 효율 거짓 라벨** — 80 PLUS 미인증 / 위장. Cybenetics / Hardware Busters / Cultists Network 리뷰 참고.
4. **VRM 발열 무시** — 보드 위 VRM heatsink 가 90 °C 넘으면 VRM throttling → CPU clock ↓.
5. **다른 PSU 의 modular 케이블 혼용** — 같은 모양이라도 핀 매핑 다름. 같은 PSU 의 케이블 사용.
6. **single-rail PSU 가 항상 좋다** — multi-rail 의 안전 ↑.
7. **PSU 노화 무시** — 5-10 년 된 PSU 의 capacitor aging.
8. **inrush current 무시** — 다수 PSU 동시 켜기로 breaker trip.
9. **UPS 시간 = 무한** 가정 — 부하 따라 1 분 만에 다 떨어짐. shutdown 자동화 필수.
10. **PSU 정전 보호 = 데이터 보호** 오해 — PSU 가 buffer 1 ms 정도. fsync / journaling 별도.
11. **PFC 없는 PSU 사용** — 전기 회사 패널티 + 인프라 부담.
12. **데스크탑 80 PLUS 만 보고 데이터센터에** — 데이터센터는 CRPS / 48V power 표준.

## 19. 관련

- [[power-cooling]]
- [[tdp-power]] — CPU/GPU 전력 식
- [[../bus-io/usb-thunderbolt]] — USB-PD 의 EPR 240 W
- [[../soc/datacenter-accelerators]] — AI rack 120 kW 전원
