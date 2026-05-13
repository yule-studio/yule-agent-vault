---
title: "메모리 (Memory) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, dram, sram, ddr, hbm, ecc, cxl]
---

# 메모리 (Memory) — Hub

**[[../hardware|↑ hardware]]**

> 휘발성 메인 / 캐시 메모리. 비휘발성은 [[../storage/storage]] 에서. CPU 와 메모리 사이의 latency 가 사실상 시스템 성능의 천장.

## 0. 한 줄

**프로그램과 데이터를 잠시 들고 있는 칩의 집합. 빠르고 작은 단 (캐시) ~ 느리고 큰 단 (DRAM) 의 계층.**

## 1. 메모리 계층
→ [[memory-hierarchy|메모리 계층]]

레지스터 → L1 → L2 → L3 → DRAM → NVMe → 네트워크. 단 하나 내려갈 때마다 latency 가 ~10 배씩.

## 2. SRAM 셀 (캐시)
→ [[memory-hierarchy]]

6-T cross-coupled inverter. 빠르지만 면적이 DRAM 의 ~10 배. 캐시 / 작은 임베디드 메모리에 사용.

## 3. DRAM 셀 (메인 메모리)
→ [[dram-cell|DRAM 셀 동작]]

1-T 1-C. 캐패시터 누설 → refresh 필요 (64 ms 표준). Row activate → sense amp → column → precharge.

## 4. DDR / LPDDR / HBM 진화
→ [[ddr-evolution|DDR 세대 진화]]

| 세대 | 출시 | 전압 | 표준 속도 | 채널 BW |
| --- | --- | --- | --- | --- |
| DDR4 | 2014 | 1.2 V | 3200 MT/s | 25.6 GB/s |
| DDR5 | 2020 | 1.1 V | 6400 MT/s | 51 GB/s |
| LPDDR5X | 모바일 | 1.05 V | 8533 MT/s | — |
| HBM3 | 2022 | — | per stack 1+ TB/s | GPU 옆 짧은 거리 |
| HBM3e | 2024 | — | 1.2+ TB/s | H200/Blackwell |

CXL.mem 으로 PCIe 위에 byte-addressable 메모리 풀 가능 — 2025-2026 본격 도입.

## 5. ECC / Rowhammer
→ [[ecc-rowhammer|ECC 와 Rowhammer]]

SEC-DED (64+8 bit) — 1 bit 정정 / 2 bit 감지. 서버 필수. Rowhammer = 인접 row 반복 활성화로 비트 플립 유발 (2014 발견).

## 6. DIMM 폼팩터

| 종류 | 용도 |
| --- | --- |
| UDIMM | 데스크탑 |
| RDIMM | 서버, register buffer |
| LRDIMM | 큰 용량 서버 |
| SODIMM | 노트북 |
| CAMM2 | 새 노트북 표준 |

### Rank / Bank / Channel
- **Rank** — 한 번에 64-bit 출력을 만드는 칩 묶음. 1R / 2R / 4R.
- **Bank** — DIMM 내부 독립 메모리 어레이. DDR5 = 32 bank/dimm.
- **Channel** — CPU 메모리 컨트롤러의 독립 경로. 듀얼 / 쿼드 / 옥타.
- bandwidth 는 채널 수에 정비례 — 8 채널 보드에 4 슬롯만 채우면 BW 절반.

## 7. 함정

1. **세대 혼용** — DDR4 와 DDR5 같이 못 꽂음 (물리 키).
2. **단일 채널 운영** — 듀얼 채널 보드에 1 모듈만 꽂으면 BW 절반.
3. **LPDDR 와 DDR 혼동** — 모바일용 LPDDR 은 메인보드 슬롯에 안 들어감.
4. **ECC 안 켜고 서버** — 1 bit 비트 플립이 production DB 손상.
5. **NUMA 무시** — 다른 소켓 메모리 접근은 latency 1.5–3 배.

## 8. 관련

- [[../transistor/transistor]] — SRAM/DRAM 셀의 트랜지스터 단
- [[../storage/storage]] — 비휘발성 매체
- [[../soc/soc]] — UMA / 메모리 컨트롤러 통합
- [[../../operating-system/operating-system]] — 가상 메모리 / 페이지 / 캐시
