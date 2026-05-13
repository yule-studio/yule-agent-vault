---
title: "TDP 와 전력"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, power, tdp, pl1, pl2, tau, ppt, throttling, dvfs, undervolt, dynamic-power, leakage, dvs]
---

# TDP 와 전력

**[[power-cooling|↑ 전원·쿨링]]**

> 칩의 발열 / 전력 / 클럭 의 정확한 의미. 마케팅 TDP 와 실제 PL2 / PPT 의 차이.

## 1. TDP — 정확한 의미

**Thermal Design Power**. 정격 부하에서 **쿨러가 처리해야 하는 열량 (W)**.

### 오해 1순위
> "TDP 125 W CPU 는 항상 125 W 만 먹는다."

→ **틀렸음.** TDP 는:
- 쿨링 / PSU 사이징 기준.
- ≠ 최대 전력 소비.
- ≠ 최저 전력 소비.
- 정의가 vendor 별 다르고 세대마다 다름.

### Intel 의 TDP
- 옛 (Skylake 이전) = base clock 에서 정격 부하의 평균.
- 현 (12th gen+) = PL1 (Power Limit 1) = 평균 전력. 단, PL2 까지 단기 부스트.

### AMD 의 TDP
- 더 보수적. base clock + PBO disabled 의 정격.
- PPT (Package Power Tracking) = 진짜 한계.

## 2. 동적 전력 — 식

```
P_dynamic = α · C · V² · f
```

- **α**: 스위칭 활동률 (0~1). idle 0, max load 1.
- **C**: load 캐패시턴스 (회로 디자인 고정).
- **V**: 공급 전압 (Vcc / Vcore).
- **f**: 클럭 주파수.

### V 의 2 차 의존
- 1.0 V → 0.95 V (5% ↓) 면 P ≈ V² 으로 10% ↓.
- 1.2 V → 1.0 V (17% ↓) 면 P ≈ V² 으로 30% ↓.
- **DVFS (Dynamic Voltage Frequency Scaling)** 의 본질.

### 같은 freq 라도 V 차이로 전력
- 5.0 GHz @ 1.40 V vs 5.0 GHz @ 1.25 V → 후자가 ~20% 적게 먹음.
- 같은 silicon 의 individual variation 때문.
- 좋은 chip ("golden sample") 은 낮은 V 로 안정 동작.

## 3. 누설 전력 (Static Power)

스위칭 안 해도 흐르는 전류:

| 종류 | 의미 |
| --- | --- |
| **Subthreshold leakage** | OFF 상태 V_GS < V_th 의 작은 전류. exponential 의존 V_th. |
| **Gate leakage** | 게이트 산화막을 직접 통과하는 양자 터널링. |
| **GIDL (Gate-Induced Drain Leakage)** | drain↔body 사이 누설. |
| **Junction leakage** | reverse-biased p-n 접합 누설. |

### 누설의 추세
- 7 nm 이하: 누설이 전체 전력의 **30~50%**.
- Dennard scaling 종말의 핵심 원인.
- HKMG (45 nm) / FinFET (22 nm) / GAA (3 nm) 가 점진 대응.

### 누설의 영향
- idle 에서도 발열.
- 데이터센터 의 idle 도 큰 비용.
- 전원 끄지 않으면 power saved 한 만큼 idle power 가 잡아먹음.

## 4. Intel — PL1 / PL2 / Tau / PL3 / PL4

### 매개변수

| 파라미터 | 의미 |
| --- | --- |
| **PL1 (Power Limit 1)** | 장기 평균 전력. 보통 TDP 와 같거나 같지 않음. |
| **PL2 (Power Limit 2)** | 짧은 부스트 한계. PL1 의 1.25~2.0×. |
| **Tau (τ)** | PL2 를 유지할 수 있는 시간 (s). 보통 28-56 s. |
| **PL3** | 1 ms 단위 spike. PSU 가 받아야 함. |
| **PL4** | 즉시 차단 한계. |

### 동작
1. 부하 시작 → 클럭이 PL2 까지 부스트.
2. Tau 시간 동안 PL2 유지.
3. Tau 후 → PL1 로 강제 다운.
4. 부하 ↓ → 클럭 ↓ → 전력 ↓.

### 예: Intel i9-14900K
- TDP 라벨: 125 W (PL1).
- PL2: 253 W. Tau: 56 s.
- 단일 코어 부스트 5.7-6.0 GHz, all-core 5.5 GHz 근처.

### 메인보드의 "no PL2 limit" 설정
- 일부 메인보드가 비용 줄이려고 항상 PL2 동작 → 발열 폭증.
- 운영자 모르고 default 그대로 사용 시 thermal 위험.

## 5. AMD — PPT / TDC / EDC

### 매개변수

| 파라미터 | 의미 |
| --- | --- |
| **PPT (Package Power Tracking)** | 전체 패키지 전력 한계 (W) |
| **TDC (Thermal Design Current)** | 평균 전류 한계 (A) |
| **EDC (Electrical Design Current)** | 짧은 spike 전류 한계 |
| **TjMax** | 최대 허용 junction 온도 (95-105 °C) |

### PBO (Precision Boost Overdrive)
- 위 3 매개변수 풀어줘서 더 높은 부스트.
- 사용자 OC 의 대안.
- BIOS 에서 enable.

### Curve Optimizer (AMD)
- 각 core 의 voltage curve 를 -1 ~ -30 사이 offset.
- core 별 다른 silicon 품질 ("preferred core") 활용.
- 안정성 한계까지 undervolt.

## 6. Thermal Throttling

### 동작
- 코어 / 패키지 온도가 TjMax 도달 → 클럭 자동 다운.
- 1 회 1 초 벤치 결과 ↔ 5 분 sustained 결과 차이의 이유.

| Tj | 동작 |
| --- | --- |
| < 80 °C | 정상 |
| 80-90 °C | 일부 boost 못 함 (mild throttling) |
| 90-100 °C | aggressive throttling |
| > TjMax (100-105 °C) | shutdown |

### 측정
```bash
# Linux
sensors
sudo apt install lm-sensors
sudo sensors-detect

# Intel
sudo apt install msr-tools
sudo modprobe msr
sudo rdmsr 0x19c            # IA32_THERM_STATUS

# Real-time
watch -n 1 sensors

# turbostat (Intel)
sudo turbostat --interval 1

# Apple Silicon
sudo powermetrics --samplers cpu_power,smc -i 1000

# 통합 interactive
sudo apt install s-tui
s-tui
```

## 7. GPU TGP / TBP / TDP

### 약자
- **TGP (Total Graphics Power)** — GPU 칩만.
- **TBP (Total Board Power)** — VRAM / VRM / 팬 포함 카드 전체.

### NVIDIA 의 표준
- RTX 4090: TGP 450 W = TBP 450 W (대부분 같음).
- RTX 5090: TBP 575 W.

### 데이터센터 GPU
- H100 SXM5: 700 W (configurable 350~700).
- H200: 700 W.
- B200: 1000 W.
- GB200 module: 2700 W (Grace + 2 × B200).

### Transient spike
- GPU 의 ms 단위 spike 가 정격의 1.5-2 배.
- PSU OCP margin 충분히.

## 8. CPU 별 전력 비교 (2024)

| CPU | TDP / PL1 | PL2 / boost | core 수 |
| --- | --- | --- | --- |
| Intel i9-14900K | 125 W | 253 W | 24 (8P+16E) |
| Intel Core Ultra 9 285K | 125 W | 250 W | 24 (8P+16E) |
| Intel Xeon Platinum 8592+ | 350 W | 425 W | 64 |
| AMD Ryzen 9 7950X | 170 W | 230 W PPT | 16 |
| AMD Ryzen 9 9950X | 170 W | 230 W PPT | 16 |
| AMD EPYC 9554 | 360 W | 400 W | 64 |
| AMD EPYC 9684X | 400 W | 450 W | 96 |
| AMD EPYC 9755 (Turin) | 500 W | 550 W | 128 |
| Apple M4 | (10 W idle ~) | ~25 W active | 10 |
| Apple M4 Max | (15 W idle ~) | ~95 W active | 16 |
| AWS Graviton 4 | 200 W | — | 96 |

## 9. 모바일 / 노트북 모드

### Power 모드
- **Low / Battery saver** = 낮은 PL1.
- **Balanced** = 기본 PL1.
- **Performance / Max** = PL1 ↑ + Tau ↑.

### Apple Silicon
- PL 식별이 미공개.
- sustained 와 burst 차이 작은 편.
- battery 시 자동 reduced.

### Intel Lunar Lake (2024)
- on-package LPDDR5X 로 idle power ↓.
- AI workloads 효율 ↑.

## 10. 노트북 sustained 성능 ≠ peak

### 현상
- 첫 1-2 분 burst 후 sustained 가 burst 의 50-70%.
- thermal mass + cooling 한계로.

### 측정
```bash
# 30 분 stress + 5 분 후 측정
stress-ng --cpu $(nproc) --timeout 30m

# Apple
asitop                                    # third-party
```

### vendor 의 fan 정책
- 게이밍 노트북 (ROG / Razer) = aggressive fan, burst 길게 유지.
- ultrabook (XPS / MacBook Air) = quiet, burst 짧음.

## 11. Undervolting / Underclocking

### 원리
- 같은 freq 에서 V ↓ → P ↓ (V² 의존).
- 같은 thermal 한계 안에서 더 긴 boost 유지 가능.

### 절차
1. 안정성 baseline 측정 (Cinebench, OCCT, Prime95).
2. V 또는 freq 한 단계 ↓ → 1 시간 stress test.
3. 통과 시 다시 ↓.
4. fail 시 한 단계 ↑.

### Intel — XTU (Extreme Tuning Utility)
- voltage offset 또는 fixed voltage.
- per-core / per-load level.

### AMD — Ryzen Master / PBO Curve Optimizer
- per-core voltage offset.
- 자동 best-core 탐지.

### 결과
- 데스크탑: -5 ~ -10% 전력 (정격이 보수적).
- 노트북: -15 ~ -25% 전력.
- 같은 thermal limit 에서 5-10% 더 빠름.

## 12. Overclock (OC) — 정 반대

### 안정성 trade-off
- freq ↑ → V ↑ → P ↑ → heat ↑.
- silicon 자체 한계 (V_BD = 게이트 breakdown).
- 노화 가속.

### 측정
- IPC (Instructions Per Cycle) × freq = 실효 throughput.
- 일부 워크로드는 single-thread freq-bound, 일부는 multi-thread 또는 메모리 bound.

### OC 의 한계 — silicon lottery
- 같은 model 도 chip 별 quality 차이.
- "golden sample" = 낮은 V 로 높은 freq.

## 13. CPU 의 전원 상태 (P-states / C-states)

### P-states (Performance states)
- P0 = 최대.
- P1, P2, ... = 단계적 freq + V ↓.
- OS / firmware 가 transition.

### C-states (Idle states)
- C0 = active.
- C1 = halt.
- C2 = stop clock.
- C3 = sleep cache.
- C6 = deep sleep (cache off).
- C7+ = package C-states (전체 package power off).

### Linux governor
```bash
cpupower frequency-info
sudo cpupower frequency-set -g performance
sudo cpupower frequency-set -g powersave

# CPU의 현재 freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# 자세히
sudo turbostat --interval 1
```

### Linux schedutil governor (현 default)
- 부하 추적 후 freq 동적.
- CFS scheduler 와 통합.

## 14. AMD 의 c-state — CPPC

- ACPI CPPC (Collaborative Processor Performance Control).
- OS 가 desired performance 를 hint 로 전달.
- CPU 가 자체 결정.
- 더 fine-grained.

## 15. Apple Silicon 의 전력 관리

### 특이점
- 별도 P-core (high perf) + E-core (efficiency).
- OS scheduler 가 workload 따라 routing.
- E-core 의 idle power 매우 낮음 (수 mW).

### SoC-level Power Gating
- 사용 안 하는 block (modem, NPU, GPU) 자동 off.
- iPhone idle 시 매우 적은 전력.

### macOS 의 측정
```bash
sudo powermetrics --samplers cpu_power,gpu_power,thermal -i 1000
```

## 16. 데이터센터 전력 / 효율

### PUE (Power Usage Effectiveness)
- 데이터센터 총 전력 / IT 장비 전력.
- 1.0 이 ideal (불가능).
- typical: 1.4-2.0.
- hyperscale 최고: **1.1-1.2** (Google, Meta).

### 효율 개선 방법
1. Liquid / immersion cooling — 풍랭 1.4 → liquid 1.1.
2. Hot aisle / cold aisle.
3. Free cooling (외부 공기).
4. 380V/415V DC 전원 (변환 ↓).
5. 발전기 / 배터리 / UPS 효율.

### Carbon footprint
- AI 학습 1 회 (GPT-4 급): ~수십 톤 CO₂.
- Hyperscale 의 renewable 비중 ↑ (Google 24/7 carbon-free 목표).

## 17. 함정

1. **"TDP 가 최대"** — 위 §1 의 가장 큰 오해. PL2 / PPT 까지 본 쿨링 / PSU.
2. **마더보드의 "no PL2 limit" 설정** — 비용 줄이려고 항상 PL2 로 동작 → 발열 폭증.
3. **PL1 = TDP 가정** — Intel 은 다름. 마더보드가 PL1 을 임의로 올려놓은 경우 흔함.
4. **GPU TGP 만 보고 PSU 사이징** — transient spike 2× 까지.
5. **노트북 sustained 성능 평가 부재** — 1 분 후 sustained 가 burst 의 50-70%.
6. **undervolting 후 long-term 불안정** — 통과한 stress test 후에도 가끔 crash. 마진 두기.
7. **fan curve 너무 quiet** — 50 °C 부터 50% fan 같은 보수적 설정이 thermal throttling 유발.
8. **PSU sleep mode 의 PL3 미충족** — 노트북에서 sleep 후 wake 시 spike 못 받음.
9. **모바일의 Cinebench 결과 신뢰** — 5 분 후 sustained 가 1/3.
10. **PUE 만 보고 효율 평가** — IT 부하 자체 (server 효율, virtualization 비율) 가 더 중요.
11. **C-states disable 후 latency 좋아짐** 가정 — idle power 폭증. server 의 worth ↓.
12. **GPU 가 PL2 max 항상 가능** — 실 사용 중 thermal / PSU limit 으로 떨어짐.

## 18. 측정 / 진단 — 종합

```bash
# Linux 전력 / 온도
sensors
sudo turbostat --interval 1                      # Intel
sudo powerstat -d 1                              # 전체 시스템
sudo apt install powertop
sudo powertop                                    # interactive

# 자세한 stress + 측정
stress-ng --cpu $(nproc) --timeout 5m
mprime                                           # Prime95
sudo apt install y-cruncher

# GPU
nvidia-smi --query-gpu=power.draw,temperature.gpu,utilization.gpu --format=csv -l 1
rocm-smi --showpower --showtemp

# macOS Apple Silicon
sudo powermetrics --samplers cpu_power,gpu_power,smc -i 1000
asitop                                           # third-party top-like

# 전체 시스템 (콘센트)
kill-a-wat                                       # 가정용
ipmitool dcmi power reading                      # 서버

# 데이터센터 PUE
# 일반적 SCADA / BMS 시스템에서.
```

## 19. 관련

- [[power-cooling]]
- [[psu-vrm]] — PSU 의 전력 공급
- [[../transistor/mosfet-cmos]] — 동적 / 누설 전력의 트랜지스터 단
- [[../transistor/process-node-evolution]] — Dennard scaling 의 종말
- [[../soc/soc-anatomy]] — Apple Silicon 의 전력 관리
- [[../soc/datacenter-accelerators]] — GPU 700-1000 W 시대
