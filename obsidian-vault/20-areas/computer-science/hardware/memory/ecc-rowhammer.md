---
title: "ECC 와 Rowhammer"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, ecc, sec-ded, rowhammer, chipkill, bit-flip]
---

# ECC 와 Rowhammer

**[[memory|↑ 메모리]]**

## 1. 비트 플립 — 왜 일어나나

1. **우주선 (cosmic ray)** — alpha 입자가 셀의 전하 상태를 뒤집음. 1 TB 메모리에서 일 평균 ~10 회 보고.
2. **알파 입자** — 패키지 안 트레이스 양의 자연 방사성.
3. **노화 (aging)** — 캐패시터·트랜지스터 열화.
4. **Rowhammer** — 의도적 / 비의도적 인접 row 반복 활성화.
5. **온도 / 전압 마진** — 운영 한계 근처에서 미세한 오류.

## 2. ECC (Error Correcting Code)

### SEC-DED (Single Error Correct, Double Error Detect)

64-bit 데이터에 8-bit ECC 부착 → 72-bit 워드.

- 1 bit 오류 → 자동 정정.
- 2 bit 오류 → 감지만 (정정 불가, OS 에 fatal report).
- 3 bit 이상 → 검출 못할 수도 있음.

Hamming code 변종. 패리티 비트를 여러 위치에 분산해 어느 비트가 깨졌는지 식별 가능.

### Chipkill / Advanced ECC

- 한 DRAM 칩 (8-bit 등) 전체가 죽어도 살아남는 ECC.
- 데이터를 여러 칩에 분산 + Reed-Solomon 류 강한 코드.
- IBM POWER, AMD EPYC, Intel Xeon 의 강 ECC 모드.

### on-die ECC (DDR5)

- DDR5 DRAM 칩 내부 자체 ECC.
- DIMM ↔ 메모리 컨트롤러 사이 통신은 일반 DDR5 라도 ECC 가 아님.
- 진짜 end-to-end ECC 는 ECC RDIMM 필수.

## 3. ECC 가 잡지 못하는 것

- **silent data corruption (SDC)** — ECC syndrome 이 같은 모양으로 발생 (확률 매우 낮음).
- **multi-bit 동시 오류** — Chipkill 이 아니면 검출만 가능.
- **버스 / 컨트롤러 오류** — DIMM 외부 오류는 별도 보호 필요.

→ 미션 크리티컬은 **ECC + checksum (e.g. ZFS scrub) + replication** 다층 방어.

## 4. Rowhammer

### 메커니즘

- DRAM row 를 짧은 시간에 수십만 번 activate/precharge.
- 인접 row 의 캐패시터에 누설 → 비트 플립.
- 2014 (Carnegie Mellon, Yoongu Kim et al.) 첫 시연 — 보안 취약점.

### 공격 형태
- **Flip Feng Shui (2016)** — 동일 DRAM row 에 page table 을 떨어뜨려 권한 상승.
- **RowPress (2023)** — 더 적은 activate 로 비트 플립 유발.
- **Half-Double (2021)** — 인접 row 가 아닌 2 칸 떨어진 row 까지 영향.

### 대응
- **TRR (Target Row Refresh)** — DRAM 컨트롤러가 의심 row 의 인접 row 를 추가 refresh. DDR4 의 표준 대응.
- **RFM (Refresh Management, DDR5)** — 컨트롤러가 명시적 refresh 명령 가능.
- **PARFM** — Probabilistic Adjacent Row Refresh.
- **ECC 보조** — 단일 비트 플립은 정정. 그러나 ECC 만으로는 부족 — 다중 플립 가능.

## 5. ECC 메모리가 필요한 경우

- 서버 / 데이터센터 — 거의 모든 경우.
- 워크스테이션 (CAD, ML 학습 호스트).
- 금융 / 미션 크리티컬 / 의료.
- 장기 가동 (1 년 이상 같은 데이터 유지) DB.

ECC 불필요한 경우:
- 게이밍 / 일반 데스크탑 (가성비 우선).
- 일회성 작업 / 매번 reboot.

## 6. 측정 / 진단

```bash
# Linux EDAC (Error Detection And Correction)
sudo apt install edac-utils
edac-util -r              # CE / UE 카운트
journalctl -k | grep -i edac

# IPMI / BMC
ipmitool sel list         # 이벤트 로그
ipmitool sdr type 'Memory'

# DRAM 모듈 상세
sudo dmidecode -t memory
```

ECC 카운트 (`Corrected Errors`) 가 폭증하면 모듈 / 슬롯 fail 임박 신호.

## 7. 함정

1. **on-die ECC 만 보고 안심** — DDR5 의 on-die ECC 는 메모리 컨트롤러 ↔ DIMM 사이 통신을 보호하지 않음.
2. **ECC 카운트 무시** — CE (Corrected Errors) 가 늘면 곧 UE (Uncorrected) 가 옴.
3. **Rowhammer 가 옛 일이라는 가정** — 새 변종이 매년 보고됨.
4. **소비자용 보드에 ECC RDIMM 끼우기** — CPU / chipset 이 모두 ECC 를 지원해야 효과.

## 8. 관련

- [[memory]]
- [[dram-cell]]
- [[ddr-evolution]]
- [[../../security-theory/security-theory]] — 사이드채널·Rowhammer 공격 모델
