---
title: "컴퓨터 구조 (Computer Architecture)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T13:30:00+09:00
tags:
  - computer-architecture
  - cpu
  - cache
  - isa
  - pipeline
---

# 컴퓨터 구조 (Computer Architecture)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**프로세서의 명령어, 메모리, I/O 의 조직 + 동작 원리**. 1945 Von Neumann 구조가
오늘날의 컴퓨터의 토대.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1945 | Von Neumann 보고서 — 저장 프로그램 컴퓨터 |
| 1971 | Intel 4004 — 첫 마이크로프로세서 |
| 1978 | Intel 8086 — x86 의 시작 |
| 1985 | MIPS R2000 — RISC |
| 1991 | ARM6 |
| 2005 | Multi-core 가 메인스트림 |
| 2010 | ARM Cortex-A9 — 모바일 정복 |
| 2020 | Apple M1 — ARM 의 PC 진입 |
| 2024 | RISC-V 부상 |

---

## 3. Von Neumann vs Harvard

### 3.1 Von Neumann

- 명령어 + 데이터를 같은 메모리
- 버스 1 개 (병목)
- 대부분 현대 CPU

### 3.2 Harvard

- 명령어 / 데이터 메모리 분리
- 동시 접근 가능
- DSP, 임베디드

### 3.3 Modified Harvard

- L1 캐시 단계에서만 분리 (L1i, L1d)
- L2 이상은 통합
- x86, ARM 모두 이 방식

---

## 4. CPU 구성요소

### 4.1 ALU (Arithmetic Logic Unit)
정수 / 비트 연산.

### 4.2 FPU (Floating-Point Unit)
부동소수점.

### 4.3 Control Unit
명령어 해석 + 신호 생성.

### 4.4 Register
- **범용** — AX/BX/RAX (x86), x0-x30 (ARM)
- **특수** — PC, SP, FLAGS, IP

### 4.5 Cache (L1/L2/L3)
[[../data-structure/arrays-and-strings/arrays-and-strings|arrays-and-strings]] 의 메모리 계층 표 참조.

### 4.6 MMU (Memory Management Unit)
가상 → 물리 주소 변환, TLB.

### 4.7 PCI / 메모리 컨트롤러
외부 장치 / RAM 통신.

---

## 5. 명령어 집합 (ISA)

### 5.1 RISC vs CISC

| 기준 | RISC | CISC |
| --- | --- | --- |
| 명령어 | 단순 / 균일 길이 | 복잡 / 가변 |
| 주소 지정 | 적음 | 많음 |
| 레지스터 | 많음 | 적음 |
| 파이프라인 | 쉬움 | 복잡 |
| 예 | ARM, MIPS, RISC-V | x86 |

x86 도 내부에선 micro-op (μop) 으로 RISC 화.

### 5.2 주요 ISA

| ISA | 비트 | 사용 |
| --- | --- | --- |
| **x86-64** (AMD64) | 64 | 데스크톱, 서버 |
| **ARM64** (AArch64) | 64 | 모바일, 서버, M1 |
| **RISC-V** | 32/64 | 오픈, 임베디드 |
| **MIPS** | 32/64 | 라우터, 임베디드 |
| **POWER** | 64 | IBM 서버 |

### 5.3 명령어 종류

- **데이터 이동** — MOV, LOAD, STORE
- **산술** — ADD, SUB, MUL, DIV
- **논리** — AND, OR, XOR, NOT
- **비교 / 분기** — CMP, JMP, JE, JNE
- **함수 호출** — CALL, RET
- **시스템** — SYSCALL, INT

### 5.4 어셈블리 예 (x86-64)
```asm
section .data
    msg db "Hello", 0xA
    len equ $ - msg

section .text
    global _start
_start:
    mov rax, 1          ; syscall: write
    mov rdi, 1          ; fd: stdout
    mov rsi, msg
    mov rdx, len
    syscall
    
    mov rax, 60         ; syscall: exit
    xor rdi, rdi
    syscall
```

---

## 6. 파이프라인 (Pipelining)

### 6.1 5 단계 클래식 RISC

```
IF → ID → EX → MEM → WB
1 명령어/사이클 (이상적)
```

- **IF** Instruction Fetch
- **ID** Instruction Decode
- **EX** Execute
- **MEM** Memory access
- **WB** Write Back

### 6.2 Hazard (위험)

- **Structural** — 자원 충돌
- **Data** — 결과 의존 (Forwarding 으로 완화)
- **Control** — 분기 (Branch Prediction)

### 6.3 Branch Prediction

- **Static** — 항상 taken / not taken
- **Dynamic** — 2-bit saturating counter
- **Tournament** — 여러 예측기 조합
- **Neural** — Intel/AMD 최신

빗나가면 파이프라인 flush — 10-20 사이클 손실.

### 6.4 슈퍼스칼라 / Out-of-Order

- 사이클 당 2+ 명령어 실행
- ROB (ReOrder Buffer), Reservation Station
- 의존성 없는 명령어 먼저 실행

### 6.5 SIMD

- Single Instruction Multiple Data
- AVX (x86), NEON (ARM), SVE (ARM 차세대)
- 1 명령어로 4-16 데이터 처리

---

## 7. 캐시 (Cache)

### 7.1 캐시 구조

```
[Tag | Set Index | Offset]   ← 주소 분해
```

- **Direct-Mapped** — Set 당 1 line
- **N-Way Set Associative** — Set 당 N line
- **Fully Associative** — 어디든 — TLB

### 7.2 캐시 라인

- 보통 64 바이트
- 한 byte 읽어도 64 바이트 가져옴

### 7.3 일관성 (Cache Coherence)

- **MESI** 프로토콜
  - Modified, Exclusive, Shared, Invalid
- 멀티코어에서 캐시 일관성 보장
- **False sharing** — 다른 코어가 같은 라인의 다른 변수 갱신 → 성능 저하

### 7.4 Hit / Miss

- **Compulsory** miss — 처음 접근
- **Capacity** miss — 캐시 가득
- **Conflict** miss — 같은 set 으로 매핑

### 7.5 Write 정책

- **Write-through** — 즉시 메모리도
- **Write-back** — Dirty bit, 나중에

---

## 8. 메모리 계층 / 가상 메모리

[[../operating-system/operating-system#7 메모리 관리]] 참조.

### 8.1 TLB (Translation Lookaside Buffer)
- 페이지 테이블 캐시
- 64-1024 엔트리
- TLB miss → 페이지 워크 (수십 사이클)

### 8.2 메모리 계층 다시
```
Register     <  1 ns
L1 cache     ~  1 ns    32-64 KB
L2 cache     ~  4 ns    256 KB - 1 MB
L3 cache     ~ 10 ns    4-64 MB
RAM (DDR5)   ~100 ns    GBs
NVMe SSD     ~100 µs
HDD          ~ 10 ms
Network      ~100 ms
```

---

## 9. 멀티코어 / 동시성 하드웨어

### 9.1 SMP (Symmetric Multi-Processing)
같은 메모리 공유 — Intel/AMD CPU.

### 9.2 NUMA (Non-Uniform Memory Access)
각 코어가 자기 가까운 메모리 빠름.
서버 / Apple M2 Ultra 등.

### 9.3 SMT (Simultaneous Multi-Threading)
- Intel **HyperThreading**
- 1 물리 코어가 2 논리 스레드
- Idle 자원 활용 (분기 / 메모리 대기)

### 9.4 Memory Ordering

- **TSO** (Total Store Ordering) — x86
- **Weak** ordering — ARM, POWER
- `mfence`, `lock` prefix 등 메모리 배리어

---

## 10. GPU / 가속기

### 10.1 GPU

- 수천 스레드 동시 (SIMT)
- CUDA (NVIDIA), ROCm (AMD), Metal (Apple)
- GEMM, 컨볼루션 등에 강함

### 10.2 TPU / NPU

- 행렬 곱 전용 (Tensor Core)
- Google TPU, Apple Neural Engine, Tesla Dojo

### 10.3 FPGA

- 재구성 가능 — Verilog/VHDL
- 저지연 응용 (HFT, 비디오)

### 10.4 ASIC

- 특정 용도 칩 — Bitcoin ASIC, Tesla FSD

---

## 11. 전력 / 클록

### 11.1 P = CV²f

- **Power = Capacitance × Voltage² × Frequency**
- 클록 ↑ 는 전력 제곱 ↑
- → 멀티코어로 전환 (Moore's Law 종말 ~2005)

### 11.2 Dennard Scaling

- 1974 Dennard — 트랜지스터 작아지면 전력 밀도 일정
- 2006 경 종료 (누설 전류)

### 11.3 Dark Silicon

- 칩의 일부만 동시에 켤 수 있음 (전력 제약)
- Heterogeneous: P-core + E-core (Apple, Intel)

---

## 12. 보안 — Side Channel

### 12.1 Spectre / Meltdown (2018)

- **Speculative execution** 이용
- 권한 없는 메모리 측정 (캐시 timing)
- 패치 → 성능 5-30% 저하

### 12.2 Rowhammer

- DRAM 행을 반복 접근해 인접 비트 뒤집기
- 권한 상승 가능

### 12.3 Cache Timing

- 캐시 hit/miss 시간 차로 비밀 측정
- AES 키 추출 등 — Constant-time 코드 필수

---

## 13. 함정 / 최적화

### 함정 1 — Cache miss 무시
랜덤 메모리 접근 vs 순차 접근 10-100x 차이.

### 함정 2 — False sharing
스레드 별로 다른 캐시 라인 (`alignas(64)`).

### 함정 3 — Branch misprediction
정렬된 배열 vs 무작위 5x 차이 (Stack Overflow 사례).

### 함정 4 — SIMD 무시
컴파일러 자동 벡터화 안 되는 경우 명시.

### 함정 5 — TLB miss
큰 메모리 → Huge Page (2MB / 1GB).

### 함정 6 — NUMA 무시
멀티 소켓 서버에서 메모리 지역성.

### 함정 7 — Memory barrier 누락
lock-free 코드에서 ordering 깨짐.

### 함정 8 — Denormal float
0 근처 부동소수점 100x 느림 (FTZ/DAZ 플래그).

---

## 14. 면접 질문

1. **Von Neumann vs Harvard**.
2. **RISC vs CISC**.
3. **파이프라인 hazard 와 해결**.
4. **Branch prediction 동작**.
5. **캐시 계층과 hit/miss**.
6. **MESI 프로토콜**.
7. **TLB 와 페이지 테이블**.
8. **SIMT (GPU) vs SIMD**.
9. **Spectre / Meltdown 의 원리**.
10. **메모리 ordering (x86 vs ARM)**.

---

## 15. 학습 자료

- **Computer Organization and Design** (Patterson / Hennessy) — RISC-V 판
- **Computer Architecture: A Quantitative Approach** (Hennessy / Patterson) — 대학원
- **CSAPP** (Bryant / O'Hallaron) — Computer Systems: A Programmer's Perspective
- **Modern Microprocessors** (Lighterra) — 무료 https://www.lighterra.com/papers/modernmicroprocessors/
- **Agner Fog** 매뉴얼 — CPU 마이크로아키텍처 깊이
- **CMU 15-213** (CSAPP) 강의

---

## 16. 관련

- [[../operating-system/operating-system]] — 가상 메모리, 컨텍스트 스위치
- [[../hardware/hardware]] — 트랜지스터, 회로
- [[../compiler/compiler]] — ISA 로의 컴파일
- [[../data-structure/arrays-and-strings/arrays-and-strings]] — 캐시 라인 / 메모리 모델
- [[../computer-science|↑ computer-science]]
