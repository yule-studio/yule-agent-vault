---
title: "PSU 와 VRM"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, power, psu, vrm, atx, 80plus, 12vhpwr, drmos]
---

# PSU 와 VRM

**[[power-cooling|↑ 전원·쿨링]]**

## 1. PSU (Power Supply Unit)

ATX 표준 = 12 V 메인 + 5 V / 3.3 V / -12 V / 5 Vsb.

### 80 PLUS 인증 (효율)

| 등급 | 50% 부하 효율 |
| --- | --- |
| 80 PLUS | 80% |
| Bronze | 85% |
| Silver | 88% |
| Gold | 90% |
| Platinum | 92% |
| Titanium | 94% |

→ 데이터센터는 90%+ 필수. 같은 W 출력 대비 발열·전기료 차이 크다.

### 폼팩터
- **ATX** — 데스크탑 풀사이즈.
- **SFX / SFX-L** — 소형 (SFF / mITX).
- **TFX** — 슬림.
- **CRPS (Common Redundant Power Supply)** — 서버 hot-swap.

## 2. 모듈러

- **Non-modular** — 모든 케이블 PSU 와 일체.
- **Semi-modular** — 메인 24-pin / CPU EPS 만 고정, 나머지 분리.
- **Full-modular** — 다 분리. 케이블 깔끔.

## 3. 12V-2x6 (구 12VHPWR) — RTX 4090 / 5090

- PCIe Gen5 부속 표준. 단일 커넥터 600 W.
- 4 개 sense pin 으로 케이블·PSU 능력 협상.
- **부분 체결 시 발열 → 녹음** 사고가 2022-2024 다수 보고.
- 12V-2x6 (rev 1.1, 2024) = 핀 깊이·sense 개선 버전.

→ **끝까지 딸각 들리도록 체결.** 케이블 곡선이 커넥터 직후 너무 가파르지 않게.

## 4. PSU 사이징

```
필요 W = (CPU PL2) + (GPU TGP × 1.2) + (시스템 100~150 W)
```

예: i9-14900K PL2 253 W + RTX 4090 450 W × 1.2 + 150 W ≈ 950 W.
→ 1000 W 80+ Gold 권장 (transient spike 대응).

### Transient spike
- GPU 의 1-10 ms 단위 spike 가 정격의 2× 까지. PSU 의 transient response 가 부족하면 OCP 발동 → 시스템 셧다운.
- 게이밍 GPU 는 항상 OCP 마진 큰 PSU 추천.

## 5. VRM (Voltage Regulator Module)

CPU 가 12 V 를 직접 못 받아서, 메인보드 위 VRM 이 12 V → 1.0–1.4 V (Vcore) 변환.

### Phase
- **multi-phase 토폴로지** — N 개 phase 가 시간 분산해서 전류 공급.
- 각 phase = MOSFET + inductor + capacitor.
- phase 가 많을수록 발열 분산 / ripple ↓ / transient response ↑.

### Component
- **High-side MOSFET** — 12 V → switch node.
- **Low-side MOSFET** — switch node → GND.
- **Driver** — gate 신호 생성.
- **DrMOS / SmartPower stage** — 위 셋을 한 칩에 통합.
- **PMIC / Controller** — phase 사이 조율.

### 데스크탑 / 서버 차이
- 데스크탑 8-phase (X670E 보드) 가 250 W PL2 처리.
- 서버 EPYC = phase 30+ 으로 500 W TDP.

## 6. PSU 의 protection

- **OCP (Over Current Protection)** — 전류 한계 초과 → 차단.
- **OVP (Over Voltage)** — 12 V 가 12.6 V 초과.
- **UVP (Under Voltage)** — load 폭증으로 전압 sag.
- **OPP (Over Power)** — 총 W 한계.
- **OTP (Over Temperature)** — 내부 thermistor.
- **SCP (Short Circuit)**.

데이터센터 PSU 는 redundancy 2N 또는 N+1.

## 7. 측정 / 진단

```bash
# Linux
sensors                              # CPU / VRM / GPU 온도 + 전압
sudo dmidecode -t 39                 # Power Supply Information
ipmitool sensor                      # 서버 / 워크스테이션

# 부하 테스트
stress-ng --cpu N --vm M --timeout 60
fio + memtester + GPU 부하 동시
```

## 8. 함정

1. **PSU 정격 ≈ 시스템 W 일 때 transient spike 로 셧다운** — 마진 30%+ 두기.
2. **12V-2x6 부분 체결** — 녹는 사고. 끝까지 체결 + 곡선 완만.
3. **저가 PSU 의 효율 거짓 라벨** — 80 PLUS 미인증 / 위장. Cybenetics / PSU 리뷰 (Hardware Busters, Cultists Network) 참고.
4. **VRM 발열 무시** — 보드 위 VRM heatsink 가 90 °C 넘으면 VRM throttling → CPU clock ↓.
5. **다른 PSU 의 modular 케이블 혼용** — 같은 모양이라도 핀 매핑 다름. **반드시 같은 PSU 의 케이블** 사용.

## 9. 관련

- [[power-cooling]]
- [[tdp-power]]
- [[../soc/datacenter-accelerators]] — 데이터센터 rack 전원
