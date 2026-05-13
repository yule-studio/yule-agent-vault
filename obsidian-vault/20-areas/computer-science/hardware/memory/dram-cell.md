---
title: "DRAM 셀 동작"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, dram, refresh, sense-amp, row, column, bank]
---

# DRAM 셀 동작

**[[memory|↑ 메모리]]**

## 1. 1-T 1-C 셀

- 1 개 트랜지스터 (액세스 게이트) + 1 개 캐패시터.
- 캐패시터에 전하 있으면 1, 없으면 0.
- 단순해서 밀도 ↑. 반면 캐패시터 누설 → **refresh** 필요.

```
   word line
       │
      ─┴─       ← 액세스 트랜지스터
       │
       │ ── bit line
       │
      ───       ← 캐패시터
       │
      GND
```

## 2. 동작 순서

```
주소 도착
   │
   ▼
Activate (ACT, tRCD)  ← row 선택 / sense amp 가 행 펼침
   │
   ▼
Read / Write (CL)     ← column 선택 / data 라인 입출력
   │
   ▼
Precharge (PRE, tRP)  ← 다음 row 위해 sense amp 닫기
```

- **tRCD (Row to Column Delay)** — Activate → Read 가능 까지.
- **CL (CAS Latency)** — Read 명령 → 데이터 도착 까지.
- **tRP (Row Precharge)** — Precharge → 다음 Activate 가능 까지.
- **tRC = tRCD + tRP** — 최소 row 사이클.

CL40 @ 6000 MT/s → 한 사이클 = 1/(6000/2 MHz) = 0.333 ns. CL40 × 0.333 = ~13.3 ns 실 시간.

## 3. 왜 read 가 destructive 인가

- 캐패시터의 전하가 sense amp 로 끌려나가면서 본래 row 가 0/1 인지 비교.
- 비교 과정에서 캐패시터 전하가 거의 0 으로 → **자동 rewrite** 필요.
- 그래서 row activate 는 한 row 전체를 sense amp 로 펼치는 큰 작업.

## 4. Refresh

- 캐패시터 누설 → 시간이 지나면 0/1 구분 불가.
- 표준 **64 ms refresh window** (DDR4) 안에 모든 row 를 한 번씩 activate.
- 8192 row → 7.8 μs 마다 1 row refresh.
- DDR5 는 **fine-grained refresh** + same-bank / all-bank refresh 옵션.

## 5. Bank / Bank Group / Rank / Channel

| 단위 | 개수 (DDR5) | 의미 |
| --- | --- | --- |
| Bank | 32 / DIMM | 독립 메모리 어레이. 동시 activate 가능. |
| Bank group | 8 / DIMM | bank 묶음. 같은 group 안 연속 access 는 더 느림. |
| Rank | 1R / 2R / 4R | DIMM 위 칩 묶음. 한 워드 (64-bit) 출력 단위. |
| Channel | 2~12 / CPU | CPU 메모리 컨트롤러의 독립 경로. |

bandwidth 가 채널 수에 정비례. 8 채널 EPYC 보드에 4 슬롯만 채우면 BW 절반.

## 6. NUMA + DRAM

듀얼 소켓 서버 / Apple M Pro/Max/Ultra:
- 각 소켓 / 다이가 자기 DRAM 을 가짐.
- 다른 소켓 메모리 접근은 **cross-socket fabric** 통과 → latency 1.5–3 배, BW 제한.
- 자세히 → [[../soc/soc-anatomy]] 의 NUMA 절.

## 7. HBM (3D 적층 DRAM)

DRAM 다이 4–12 층을 TSV (Through-Silicon Via) 로 잇고 1024-bit 광폭 버스로 GPU 옆에 패키지.

| 세대 | per-pin (Gbps) | per-stack BW | 대표 |
| --- | --- | --- | --- |
| HBM2e | 3.6 | 460 GB/s | A100 |
| HBM3 | 6.4 | 819 GB/s | H100 (80 GB, 3.35 TB/s) |
| HBM3e | 9.6 | 1228 GB/s | H200/B100 (192 GB, 8 TB/s) |
| HBM4 | 16+ | 2 TB/s+ | (예정 2026~) |

## 8. CXL.mem

PCIe 5.0+ 위에 캐시-일관성 프로토콜.
- **CXL.io** — PCIe 호환.
- **CXL.cache** — 가속기가 호스트 메모리 캐시-일관성 read/write.
- **CXL.mem** — 호스트가 가속기 / 메모리 익스팬더의 메모리를 byte-addressable 로 사용.

2025-2026 본격 도입. DRAM 가격 폭등 + AI 학습 메모리 부족의 대응책.

## 9. 함정

1. **CL 수치만 비교** — CL ÷ MT/s × 2000 을 ns 로 환산해 비교해야 의미 있음.
2. **bank 충돌** — 같은 bank 안에서 연속 access → activate → precharge → activate 가 직렬로.
3. **single rank vs dual rank** — 듀얼 랭크가 보통 더 빠름 (bank parallelism ↑).

## 10. 관련

- [[memory]]
- [[memory-hierarchy]]
- [[ddr-evolution]]
- [[ecc-rowhammer]]
- [[../../operating-system/operating-system]] — 가상 메모리, huge page
