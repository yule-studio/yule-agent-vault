---
title: "Page Table — 페이지 테이블"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:40:00+09:00
tags:
  - operating-system
  - memory
  - page-table
---

# Page Table

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 4-level x86-64 / 페이지 권한 |

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

가상 주소 → 물리 주소 매핑 자료구조. CPU 의 **MMU** 가 하드웨어로 따라간다.

---

## 2. 페이지

기본 단위 = **4 KB** (대부분 OS).

- 가상 주소: 페이지 번호 + offset
- 물리 주소: 프레임 번호 + offset (offset 동일)

```
[VPN (Virtual Page Number) | offset]
        ↓ 페이지 테이블
[PFN (Physical Frame Number) | offset]
```

---

## 3. x86-64 4-Level Paging

```
가상 주소 (48 bit, canonical):
  PML4 (9 bit) | PDPT (9 bit) | PD (9 bit) | PT (9 bit) | offset (12 bit)
```

- 4 단계 (PML4 → PDPT → PD → PT)
- 각 단계 = 9 bit = 512 entry / page (4KB / 8byte)
- 4 × 9 = 36 bit + offset 12 bit = 48 bit
- 한 프로세스 = 2^48 = 256 TB 가상

### 3.1 5-Level (Intel Ice Lake+)

```
PML5 추가 → 57 bit 가상 = 128 PB
```

---

## 4. 페이지 테이블 엔트리 (PTE)

```
[PFN (40 bit) | flags (12 bit)]
```

flags:
| Bit | 의미 |
| --- | --- |
| P (Present) | 페이지 RAM 에 있나 |
| R/W | 쓰기 가능 |
| U/S | User / Supervisor |
| A (Accessed) | CPU 가 접근했나 (LRU 근사) |
| D (Dirty) | 쓰기됐나 (write back) |
| PCD/PWT | 캐시 정책 |
| G | Global (TLB 안 flush) |
| NX | No-eXecute |

---

## 5. CR3 — 페이지 테이블 루트

x86-64 의 `CR3` 레지스터 = 현재 프로세스의 PML4 의 물리 주소.

context switch 시 다른 프로세스로 바꾸면:
- CR3 변경
- TLB flush (PCID 없으면)

---

## 6. 페이지 테이블 자체의 크기

```
1 프로세스 4 GB 가상 사용 + 4 KB 페이지 → 
PT 1 GB = 200만 entry = 16 MB

하지만 4-level 의 lazy 채움 → 실제 사용 가상만 PT 할당
```

→ 매우 큰 가상 공간이라도 PT 자체는 작다 (수십 KB ~ MB).

---

## 7. Huge Page

자세히 → [[huge-pages]]

- 2 MB 페이지 (PD 까지만, PT skip)
- 1 GB 페이지 (PDPT 까지만)

→ TLB miss ↓ + PT 크기 ↓.

---

## 8. Page Walk

CPU 가 TLB miss 시 메모리 4 번 접근 (4-level). ~ 100 ns × 4 = 400 ns 비용.

→ **TLB 가 hit 하면 0 cycle**, miss 면 4 번 메모리 read.

자세히 → [[tlb]]

---

## 9. Inverted Page Table (옛)

PFN → VPN 의 반대 방향. RAM 크기에 비례. 다중 프로세스 시 PID 추가.

PowerPC / IA-64 등 옛 사용. x86-64 = forward (multi-level).

---

## 10. Software TLB / Soft Page Table

일부 RISC-V / MIPS / SPARC = TLB miss 가 trap → 커널이 SW 로 채움.
대부분 모던 = HW page walker.

---

## 11. Process 간 공유

같은 물리 페이지를 여러 프로세스의 페이지 테이블이 가리키면 → 공유.

- read-only code / lib
- shared memory (shmget / mmap MAP_SHARED)
- CoW (fork)

---

## 12. KASLR + KPTI (Meltdown 대응)

KPTI = Kernel Page Table Isolation:
- 커널 영역의 매핑을 사용자 모드에서 보이지 않게
- 사용자 ↔ 커널 전환 시 CR3 변경
- syscall 비용 ↑ (30%)

PCID 사용 시 일부 mitigation.

---

## 13. /proc/$PID/pagemap

각 가상 페이지 → PFN 매핑 (root 만).

```bash
# /proc/$PID/maps 의 가상 → /proc/$PID/pagemap 의 64 bit / page
```

디버그 / NUMA 분석 / DRAM 위치 추적 도구.

---

## 14. 페이지 fault 와의 관계

PTE 의 P bit = 0 → 접근 시 page fault → 커널이:
- 디스크에서 swap-in
- COW 복사
- 새 페이지 할당 + zero-fill
- 또는 SIGSEGV (잘못된 접근)

자세히 → [[page-fault]]

---

## 15. 함정

### 15.1 PT 크기 무시
거대 가상 mmap (TB) 도 가상만 — PT 도 lazy. 실제 page touch 가 비싼 부분.

### 15.2 NX off 가정
모던 OS 는 NX on. 데이터 영역 실행 = SIGSEGV.

### 15.3 4KB 페이지 만 가정
Huge Page 환경에선 TLB / PT 다름.

### 15.4 cross-process pointer 공유
가상 주소 의미 X (각 프로세스 다름). 공유는 shm 또는 file mmap.

### 15.5 ASLR + 절대 주소 가정
실패. relative / PIC.

### 15.6 Hugepage / THP 의 fragmentation
긴 운영 후 transparent huge page 의 split. defrag.

---

## 16. 학습 자료

- **OSTEP** Ch. 18-20
- **Intel SDM Vol. 3, Ch. 4** (Paging)
- **What Every Programmer Should Know About Memory** — Drepper

---

## 17. 관련

- [[tlb]]
- [[page-fault]]
- [[huge-pages]]
- [[memory]] — Memory hub
