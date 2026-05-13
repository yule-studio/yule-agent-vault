---
title: "ECC 와 Rowhammer"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, ecc, sec-ded, hamming, bch, chipkill, rowhammer, flip-feng-shui, rowpress, half-double, trr, rfm]
---

# ECC 와 Rowhammer

**[[memory|↑ 메모리]]**

> 메모리 안정성의 두 축 — ECC 가 비트 플립을 자동 정정 / 검출하고, Rowhammer 는 그 비트 플립을 의도적으로 유발하는 공격.

## 1. 왜 비트 플립이 일어나는가

| 원인 | 빈도 | 의미 |
| --- | --- | --- |
| **Cosmic ray (우주선)** | 1 TB 메모리 당 일 평균 ~10 회 | 고에너지 입자가 cell 의 전하 상태 뒤집음. 고도가 높을수록 ↑ (산악 / 비행기). |
| **알파 입자** | 변동 | 패키지 트레이스 / 솔더의 자연 방사성. |
| **노화 (aging)** | 시간 누적 | 캐패시터 / 트랜지스터 열화. P/E cycle. |
| **Rowhammer** | 의도적 | 인접 row 반복 활성화로 cell 누설. |
| **온도 / 전압 마진** | 운영 한계 근처 | 미세 오류. |
| **습기 / 진동** | 산업 환경 | 신호 무결성. |

Google 의 2009 데이터 (DRAM Errors in the Wild): 평균 1 DIMM 당 연 1 회 correctable error 발생. uncorrectable 은 그 1/1000.

## 2. ECC (Error Correcting Code) 의 개념

오류 정정 코드는 데이터에 **redundancy** 를 추가해서 일부 비트가 손상돼도 원본을 복구.

### Hamming 거리 (Hamming distance)

- 두 코드워드 사이 다른 비트 수.
- 거리 d 인 코드 → ⌊(d-1)/2⌋ 비트 정정, d-1 비트 검출.

| Hamming 거리 | 정정 | 검출 |
| --- | --- | --- |
| 1 | 0 | 0 |
| 2 | 0 | 1 (parity) |
| 3 | 1 (Hamming code) | 2 |
| 4 | 1 | 3 (SEC-DED) |
| 7 | 3 | 6 |

## 3. SEC-DED (Single Error Correct, Double Error Detect)

가장 흔한 메모리 ECC. 64-bit 데이터 + 8-bit 패리티 = 72-bit 워드.

### Hamming code 의 기본

데이터 비트와 parity 비트를 섞어서 배치:

- 위치 1 (= 2⁰): parity P1.
- 위치 2 (= 2¹): parity P2.
- 위치 3: data.
- 위치 4 (= 2²): parity P4.
- 위치 5: data.
- ... 8, 16, 32 도 parity.

각 parity = 자기 위치를 포함하는 모든 위치의 비트 XOR.

읽을 때 모든 parity 다시 계산 → **syndrome** (parity 들의 XOR) 이 0 이면 OK, 아니면 syndrome 값이 손상된 비트 위치를 가리킴.

### SEC-DED 의 추가 패리티

- Hamming code 위에 **overall parity** 한 비트 더.
- 2 비트 오류 시 syndrome ≠ 0 + overall parity = 0 → **검출만** (정정 불가).
- 3+ 비트는 검출 못 할 수도 있음.

```
72 bit = 64 (data) + 8 (parity)
parity 1-bit 의 의미:
  - syndrome ≠ 0, overall mismatch: 1-bit error → 정정
  - syndrome ≠ 0, overall match:    2-bit error → 검출만 (DUE)
  - syndrome = 0, overall mismatch: 1-bit in overall parity → 정정
  - syndrome = 0, overall match:    OK 또는 3+ bit miss
```

## 4. Chipkill / Advanced ECC

SEC-DED 의 한계: 한 DRAM 칩 자체가 죽어버리면 (multi-bit error 동시) 정정 불가.

### Chipkill 동작
- 데이터를 여러 칩에 분산 + Reed-Solomon 류 강한 코드.
- 한 칩이 통째로 fail 해도 살아남음.
- 1990 년대 IBM 이 도입.
- 현재 EPYC / Xeon 의 강 ECC 모드.

### 종류
| 이름 | 출처 | 특징 |
| --- | --- | --- |
| **Chipkill** | IBM (1990s) | 한 chip fail 정정 |
| **Lockstep** | Intel | DIMM 쌍을 mirror 처럼 |
| **Advanced ECC** | Intel | Chipkill 의 Intel 명칭 |
| **DDDC (Double Device Data Correction)** | Intel | 2 칩 fail 까지 정정 |
| **AMD Chipkill** | AMD | EPYC 의 강 ECC |
| **SDDC (Single Device Data Correction)** | 일반 | 1 칩 fail 정정 |

## 5. DDR5 on-die ECC — DRAM 내부 자체 ECC

DDR5 DRAM 칩 내부에 자체 ECC.

### 동작
- 칩 안에서 128-bit 데이터에 16-bit ECC.
- read 시 자동 정정 (single bit).
- write 시 자동 parity 생성.

### 한계 / 오해
- **DIMM ↔ 메모리 컨트롤러 사이 통신은 ECC 가 아님**.
- 컨트롤러가 DRAM 에 잘못된 데이터 보내도 검출 못 함.
- 진정한 end-to-end ECC = **ECC RDIMM + Chipkill 지원 컨트롤러**.

## 6. ECC 가 잡지 못하는 것

| 종류 | 의미 |
| --- | --- |
| **Silent Data Corruption (SDC)** | ECC syndrome 이 우연히 valid 한 코드로 변형. 확률 매우 낮음 (10⁻¹⁵+) 지만 0 아님. |
| **Multi-bit 동시 오류** | SEC-DED 는 2-bit 검출만. Chipkill 도 동시 칩 2 개+ fail 은 못 함. |
| **버스 / 컨트롤러 오류** | DIMM 외부 오류. 별도 보호 필요. |
| **노화 누적** | EOL 가까운 셀의 marginal 동작. ECC 카운터 ↑ 가 신호. |

→ 미션 크리티컬은 **ECC + checksum (e.g. ZFS, BCD 의 page checksum) + replication + 정기 scrub**.

## 7. ECC RDIMM 의 종류

| 종류 | buffer | 용도 |
| --- | --- | --- |
| **UDIMM (Unbuffered)** | 없음 | 데스크탑 (일부 워크스테이션). 비-ECC 거의 표준. |
| **ECC UDIMM** | 없음 + ECC chip | 일부 워크스테이션 / AMD AM5 / Ryzen Pro. |
| **RDIMM (Registered)** | 명령 라인에 register | 서버. 안정성·용량 ↑. |
| **LRDIMM (Load Reduced)** | 데이터 라인까지 buffer | 큰 용량 서버. |
| **3DS RDIMM** | TSV 적층 + register | 256 GB+ enterprise DIMM. |

### Register / Buffer 의 효과
- 메모리 컨트롤러의 전기적 부하 줄임.
- 더 많은 DIMM, 더 높은 용량 가능.
- 단점: latency 1-2 ns 추가.

## 8. ECC 효용 — 실 사례

### 1. Google (2015)
- 6 년 데이터 분석 결과 메모리 오류가 사람들이 추정보다 훨씬 많음.
- 어떤 모듈은 정상보다 100× 오류율.
- ECC + monitoring 으로 fail 임박 모듈 자동 retire.

### 2. AWS EC2
- 호스트 단에서 비-ECC 메모리 안 씀.
- ECC 카운터가 임계치 넘으면 호스트 자동 deprecation.

### 3. Bitsquat (DNS 비트 플립 공격, 2011)
- 인기 도메인 (microsoft.com) 의 한 비트 flipped 도메인 (mic2osoft.com) 등록.
- DRAM 비트 플립으로 DNS lookup 이 잘못된 IP 받음 → 공격자 사이트.
- 실 환경에서 발생 확인.

## 9. Rowhammer — 이제 본격 공격

### 메커니즘

```
   row N-1   ────────── 이웃 row (victim)
   row N     ─[hammer]─ 공격 row (반복 activate)
   row N+1   ────────── 이웃 row (victim)
```

- row N 을 1 초에 수십만 번 activate / precharge.
- row N 의 word line voltage 가 인접 row 의 access transistor 에 미세한 누설 전류 induce.
- 인접 row 의 캐패시터 전하 누설 → 비트 플립.

### 2014 첫 공개 — Yoongu Kim et al. (Carnegie Mellon + Intel)
- "Flipping Bits in Memory Without Accessing Them"
- LPDDR / DDR3 에서 재현 가능 입증.
- 보안 취약점으로 인식.

### Drammer (2016)
- Android 의 Rowhammer.
- 사용자 권한 → root.

### Flip Feng Shui (2016)
- DRAM 의 같은 row 에 page table 을 의도적으로 떨어뜨려 권한 상승.
- VM escape 가능.

### RowPress (2023)
- 더 적은 activate (수만 회) 로 비트 플립.
- DRAM row 를 길게 유지하면서 인접 row 누설 가속.

### Half-Double (2021, Google)
- 인접 row 가 아닌 2 칸 떨어진 row 까지 영향.
- TRR (Target Row Refresh) 가 인접 row 만 보호하므로 우회.

### Blacksmith (2021)
- 다양한 access 패턴으로 RowHammer.
- 기존 TRR 우회.

## 10. Rowhammer 대응

### TRR (Target Row Refresh) — DDR4

- DRAM 컨트롤러가 의심 row 의 인접 row 를 추가 refresh.
- access counter 추적해서 hot row 감지.
- 단점: 구현 vendor 별 다름. Half-Double / Blacksmith 같은 우회 공격이 가능.

### RFM (Refresh Management) — DDR5

- 컨트롤러가 명시적 refresh 명령 발행 가능.
- DRAM 의 hot row 카운터 (RAA, Rolling Accumulated ACT) 가 임계치 넘으면 컨트롤러가 RFM 명령.
- 더 강한 보호 + 더 많은 throughput 영향.

### on-die ECC (DDR5)

- 단일 비트 플립은 자동 정정.
- Rowhammer 가 다중 비트 만들면 못 막음.
- 보조적 역할.

### Probabilistic Adjacent Row Refresh (PARA)

- 학술 제안 (2014).
- 매 activate 마다 확률적으로 인접 row refresh.
- 일부 칩에서 구현.

### Software 대응

- **Cache flush 빈도 감지** — Rowhammer 는 CLFLUSH 의 빈번한 사용 필요.
- **메모리 격리** — 보안 민감 영역과 일반 영역 분리.
- **Coffee Lake+ 의 Intel CAT (Cache Allocation Technology)** — 캐시 영역 격리.

## 11. ECC 메모리가 필요한 경우

### 항상 필요
- 서버 / 데이터센터.
- 금융 / 의료 / 미션 크리티컬.
- 1 년 이상 가동 DB.
- 워크스테이션 (CAD / ML 학습 호스트 / VFX).

### 권장 (워크로드 따라)
- 개발 워크스테이션 (큰 컴파일 / VM 호스트).
- 게임 서버 (성능 비결정성 줄임).
- NAS (ZFS 의 경우 거의 필수 — 비트 플립이 RAID 전체 corrupt 가능).

### 불필요
- 게이밍 / 일반 데스크탑.
- 일회성 작업.

## 12. ECC 진단 — Linux EDAC

EDAC (Error Detection And Correction) 프레임워크.

```bash
sudo apt install edac-utils
sudo modprobe sb_edac          # Intel Sandy Bridge+
sudo modprobe amd64_edac       # AMD

edac-util -r                   # 요약
# mc0: csrow0: ce=0 ue=0
# mc0: csrow1: ce=15 ue=0     ← CE 가 늘면 모듈 fail 임박

# 자세히
sudo cat /sys/devices/system/edac/mc/mc0/csrow*/ce_count
sudo cat /sys/devices/system/edac/mc/mc0/csrow*/ue_count

# 어느 슬롯 / 어느 모듈인지
sudo dmidecode -t memory | grep -B 3 -A 12 "Locator:"

# kernel 로그
journalctl -k | grep -i "edac\|mce"
sudo mcelog --client                # MCE (Machine Check Exception)
```

### IPMI / BMC (서버)
```bash
ipmitool sel list                   # 시스템 이벤트 로그
ipmitool sdr type 'Memory'
```

CE (Corrected Errors) 카운트가 폭증 → 모듈 / 슬롯 fail 임박. 보통:
- 시간당 0 ~ 수 회 = 정상.
- 시간당 10+ 회 = 모듈 교체 검토.
- UE (Uncorrected) 가 1 회라도 = 즉시 교체 + 데이터 검증.

## 13. ECC 시뮬레이션 / 검증

### memtester
- 메모리 스트레스 + 패턴 테스트.
```bash
sudo memtester 4G 5      # 4 GB 메모리를 5 번 테스트
```

### memtest86+ (BIOS / UEFI 부팅)
- 단독 부팅. OS 없이 메모리 전체 검사.
- 비트 패턴 다양 + walking ones / zeros.

### Stressapptest (Google)
- 데이터센터용. 더 realistic 한 워크로드.
```bash
sudo apt install stressapptest
sudo stressapptest -s 60 -M 8000   # 60 초, 8 GB
```

## 14. 함정

1. **on-die ECC 만 보고 안심** — DDR5 의 on-die ECC ≠ end-to-end ECC. ECC RDIMM 별도 필요.
2. **소비자용 보드에 ECC RDIMM 끼우기** — CPU + chipset + 보드 모두 ECC 지원 + RDIMM 지원해야 함. AM5 (Ryzen) 는 UDIMM 만, AM5 Pro 는 일부 ECC UDIMM.
3. **ECC 카운트 무시** — CE 가 늘면 곧 UE 가 옴. 모니터링 자동화.
4. **Rowhammer 가 옛 일이라는 가정** — 새 변종 (RowPress, Half-Double) 매년 보고.
5. **EDAC 모듈 자동 로드 가정** — 일부 배포판 / 커널은 명시 modprobe 필요.
6. **ECC 와 NUMA 혼동** — 다른 NUMA 노드의 ECC 통계는 다른 mc 디바이스.
7. **클라우드 VM 에서 ECC 보임** — 호스트는 ECC, VM 은 noecc. 클라우드 vendor 의 호스트 단 retire 정책 신뢰.
8. **memtest 한 번 통과 = 평생 OK** — 메모리는 시간에 따라 fail. 분기별 정기 monitoring.
9. **rowhammer 대응 = TRR 만** — DDR4 의 TRR 는 일부 공격 우회 가능. DDR5 + RFM 권장.
10. **uncorrectable 발생 시 reboot 만** — 데이터 검증 + 백업 비교 필요. corruption 이 디스크까지 갔을 수 있음.

## 15. 관련

- [[memory]]
- [[dram-cell]] — Rowhammer 의 셀 물리적 원인
- [[ddr-evolution]] — DDR5 on-die ECC, RFM
- [[../../security-theory/security-theory]] — Rowhammer 의 보안 모델 / DMA attack
- [[../storage/raid-levels]] — RAID 와 ECC 의 다층 방어
