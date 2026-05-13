---
title: "Page Replacement — 페이지 교체 알고리즘"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:55:00+09:00
tags:
  - operating-system
  - memory
  - replacement
---

# Page Replacement

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | LRU / Clock / 2Q / ARC |

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

RAM 가득 차서 새 페이지 적재 시 어떤 페이지를 swap-out 할지 결정.

목표: **재접근 가능성 낮은 페이지** 를 제거.

---

## 2. 알고리즘

### 2.1 Optimal (Belady)
미래에 가장 늦게 사용될 페이지. 이론적 최적, 실제 불가 (미래 모름).

### 2.2 FIFO
들어온 순서. 단순. **Belady's Anomaly** — 프레임 늘려도 fault 늘어날 수 있음.

### 2.3 LRU (Least Recently Used)
가장 오래 안 쓴 페이지. 직관적. 정확 구현 비쌈 (모든 access 갱신).

### 2.4 Clock (Second-Chance)
LRU 근사. 페이지마다 reference bit. 시계 침처럼 돌며 0 인 페이지 교체, 1 이면 0 으로 reset 후 다음.

### 2.5 NFU / LFU
사용 횟수. 옛 인기 → 새 인기 못 따라감 (aging 필요).

### 2.6 Aging
LFU + 시간 가중. Linux 의 active/inactive 가 유사.

### 2.7 Working Set (Denning)
최근 시간 윈도우 안의 페이지들.

### 2.8 2Q
- 첫 접근 → A1 (FIFO, 단기)
- A1 에서 다시 접근 → Am (LRU, 장기)
- A1 evict → Aout (메타만, 단기 기록)
- Aout 의 페이지 재접근 시 즉시 Am
- one-time scan 의 cache pollution 방지

### 2.9 LRU-K / 2 (O'Neil)
K-번째 마지막 접근 시간 기반.

### 2.10 ARC (Adaptive Replacement Cache)
T1 (recent) + T2 (frequent) + B1 + B2 (ghost). Adaptive split. ZFS / 일부 DB.

### 2.11 CAR / CART
ARC + Clock — race 없이 효율적.

---

## 3. Linux 의 페이지 교체

```
LRU list:
  Active (자주 접근)
  Inactive (옛 / 한 번 접근)

reclaim 시:
  inactive → active 로 promote (재접근)
  inactive → reclaim 후보
```

- per-zone LRU
- `kswapd` 가 백그라운드 reclaim
- `vmscan.c`

```bash
cat /proc/zoneinfo | grep -E 'nr_active|nr_inactive'
```

---

## 4. swappiness

```bash
cat /proc/sys/vm/swappiness
# 0 - 100, 기본 60
# 0 = anonymous 페이지 swap 거의 X (file cache 먼저)
# 100 = anonymous / file 동등
```

DB 서버는 보통 1 (또는 10) — swap 회피.

---

## 5. zswap / zram

```
zswap: swap 압축 (디스크 swap 전 메모리)
zram:  RAM 의 압축 블록 디바이스 → swap 으로 사용
```

→ 디스크 swap 대신 메모리 압축.

---

## 6. WSClock

Working Set + Clock 결합. 실용적 LRU 근사. 일부 OS 의 페이지 교체.

---

## 7. Application Cache 의 비슷한 알고리즘

같은 알고리즘이 응용 / DB / Redis 캐시에서도:

| 시스템 | 정책 |
| --- | --- |
| Linux page cache | active / inactive (2Q 유사) |
| Redis | LRU / LFU (sample 기반) |
| Memcached | LRU |
| PostgreSQL shared buffer | clock-sweep |
| InnoDB | LRU + midpoint insertion (2Q 유사) |
| ZFS ARC | ARC |

자세히 → [[../../database/mysql/innodb-engine#4-buffer-pool]]

---

## 8. Belady's Anomaly

```
3 frame:  fault 9
4 frame:  fault 10  ← 더 많은 frame 인데 더 많은 fault
```

FIFO 에서 발생. LRU 는 stack 속성 (more frames ≥ fewer faults).

---

## 9. Thrashing

```
RSS > 가용 RAM
→ 페이지 교체 폭증
→ CPU 의 90%+ 가 page fault 처리
→ 응답 없음
```

해결:
- 작업 축소
- swap 끄기 + OOM
- 메모리 추가

자세히 → [[swap-oom]]

---

## 10. mmap + page cache

```
read() 호출:  파일 → kernel page cache → user buffer
mmap():       파일 → kernel page cache 직접 매핑 (zero-copy)
```

→ mmap 은 page cache 와 통합 → 같은 페이지 교체 정책 적용.

---

## 11. NUMA + 교체

NUMA node 별 LRU. cross-node migration 가능 (`numa_balancing`).

---

## 12. 측정

```bash
vmstat 1
# si/so = swap in/out per second
# free / cached / buff / si / so

sar -B 1
# pgpgin/s pgpgout/s pgflt/s pgmajflt/s

pidstat -t -r 1

cat /proc/vmstat | grep -E 'pageoutrun|pgsteal|pgscan'
```

---

## 13. 함정

### 13.1 LRU "정확" 구현
모든 access 마다 list 갱신 = 비용 ↑. Clock 으로 근사.

### 13.2 One-time scan 의 cache pollution
큰 파일 1회 read → page cache 오염. `madvise MADV_DONTNEED` 또는 2Q.

### 13.3 swappiness 너무 큼
DB 의 anonymous (buffer pool) 가 swap 으로 → 성능 X.

### 13.4 Thrashing
RSS > RAM = 죽음. 모니터링.

### 13.5 Belady's Anomaly 모름
디버그 어려움 (예측 X).

### 13.6 zswap 활성 후 latency
press 평가 필요.

---

## 14. 학습 자료

- **OSTEP** Ch. 22-23
- **Understanding the Linux Virtual Memory Manager** — Gorman
- **ARC paper** — Megiddo & Modha (2003)
- **2Q paper** — Johnson & Shasha (1994)

---

## 15. 관련

- [[swap-oom]]
- [[page-fault]]
- [[memory]] — Memory hub
