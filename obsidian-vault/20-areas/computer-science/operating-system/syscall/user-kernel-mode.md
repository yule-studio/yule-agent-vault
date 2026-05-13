---
title: "User Mode vs Kernel Mode — Privilege Levels"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:50:00+09:00
tags:
  - operating-system
  - syscall
  - kernel
---

# User Mode vs Kernel Mode

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | privilege / 전환 / 비용 |

**[[syscall|↑ Syscall hub]]**

---

## 1. 한 줄

CPU 가 제공하는 **권한 레벨**. user mode = 일반 응용, kernel mode = OS 핵심. 안전성과 격리의 기반.

---

## 2. x86 의 Ring

```
Ring 0  — Kernel (모든 명령)
Ring 1  — 거의 안 씀
Ring 2  — 거의 안 씀
Ring 3  — User
```

대부분 OS = ring 0 + 3 만. 가상화 (Hyper-V) 가 Ring -1 (VMX).

ARM 의 EL (Exception Level):
- EL0 — User
- EL1 — Kernel
- EL2 — Hypervisor
- EL3 — Secure Monitor

---

## 3. 무엇이 다른가

| | User | Kernel |
| --- | --- | --- |
| 명령 | 제한 (privileged X) | 모두 |
| 메모리 | 자기 가상 공간 | 전체 + 물리 직접 |
| I/O 명령 | X | O |
| 인터럽트 | 못 disable | disable 가능 |
| CR / MSR register | X | O |
| 잘못된 동작 | 자기 만 죽음 | 전체 죽음 |

---

## 4. 전환 — User → Kernel

### 4.1 syscall (의도적)

```
asm: syscall (x86-64)
  → MSR LSTAR 에 등록된 entry 로 점프
  → kernel mode 전환
  → 결과 반환 후 sysret 으로 user 복귀
```

### 4.2 trap / exception (강제)
- Page fault
- Divide by zero
- Invalid opcode
- Breakpoint

### 4.3 interrupt (외부)
- Timer
- I/O 완료
- Inter-Processor Interrupt

---

## 5. Cost

### 5.1 직접 비용
- syscall instruction: ~100-200 cycles
- 인터럽트 / trap: ~수백 cycles

### 5.2 간접 비용
- TLB miss (CR3 / address space 변경 안 함, 그래도 일부)
- Branch predictor reset
- L1 cache cold (kernel 코드 fetch)
- **KPTI** (Meltdown 패치) → page table 두 번 변경 → ~50% 더 비쌈

### 5.3 결과
- `getpid()` ~100ns
- `read(fd, buf, 8)` ~수백 ns ~ 수 μs
- 큰 read = 디스크 비용이 더 크지만, 작은 read 많이 = syscall 누적

→ syscall batch / vectored I/O / io_uring 으로 amortize.

---

## 6. KPTI (Kernel Page Table Isolation)

Meltdown 패치 (2018):
```
User mode 에선 kernel 페이지가 page table 에 안 보임
→ syscall 시 page table 교체 → TLB flush (PCID 없으면)
```

성능:
- syscall 비용 ↑ 30-50%
- 작은 syscall heavy 워크로드 (DB, NIC) 영향 큼

mitigation:
- PCID 지원 CPU 면 일부 회피
- vDSO 로 가장 자주 호출 syscall 회피
- io_uring 으로 syscall 자체 ↓

```bash
# 비활성 (위험!)
# nopti kernel param
# 또는 sysctl
```

---

## 7. Spectre

투기적 실행 (speculative execution) 으로 권한 경계 우회 → 데이터 누출.

mitigation:
- IBRS / STIBP / IBPB MSR
- retpoline (return ↔ ret 사용)
- 컴파일 옵션 (`-mindirect-branch=thunk-extern`)

→ syscall / context switch 비용 추가.

---

## 8. vDSO — kernel 진입 회피

자주 호출되는 read-only syscall 의 코드를 user 공간 read-only 매핑:

```bash
cat /proc/$PID/maps | grep vdso
# 7ffd... [vdso]
```

함수:
- `gettimeofday`
- `clock_gettime`
- `getcpu`
- `time`

→ syscall 모드 전환 X. 매우 빠름.

---

## 9. Kernel 의 컨텍스트

### 9.1 Process context
syscall 처리 중 — current task 정보 있음 (current 매크로).

### 9.2 Interrupt context
인터럽트 핸들러 — sleep / wait 불가. spinlock 만.

### 9.3 Atomic context
인터럽트 핸들러 + softirq + tasklet + spinlock 잡힌 상태.

→ 커널 코드는 어느 context 인지 항상 인식.

---

## 10. preemption

```
CONFIG_PREEMPT_NONE       — 서버, 좋은 throughput
CONFIG_PREEMPT_VOLUNTARY  — 데스크탑 보통
CONFIG_PREEMPT            — 응답성 ↑
CONFIG_PREEMPT_RT         — 실시간 (커널도 fully preemptible)
```

자세히 → [[../scheduling/realtime]]

---

## 11. 커널 진입 / 복귀 (간략)

```
user mode (CPL=3):
  - syscall instruction
  - CPU: SS/CS/RIP/RFLAGS 저장
  - LSTAR 의 entry 로 점프
  - CPL = 0

kernel mode (CPL=0):
  - entry trampoline (KPTI: page table 교체)
  - syscall handler 진입 — dispatch table
  - 처리
  - sysret instruction

user mode 복귀:
  - register 복원
  - CPL = 3
```

---

## 12. 시스템 콜 ABI

x86-64 Linux:
```
RAX = syscall number
RDI, RSI, RDX, R10, R8, R9 = 1st-6th arg
syscall
RAX = return (또는 -errno)
```

ARM, RISC-V 등은 다름. glibc wrapper 가 ABI 추상.

---

## 13. 권한 escalation

user 가 root 가 되는 경로:
- setuid binary
- sudo
- capability set (CAP_SYS_ADMIN 등)
- kernel exploit (CVE) — 절대 X

자세히 → [[../security/permissions]]

---

## 14. 함정

### 14.1 syscall 폭증
profiling. batch / vectored / io_uring.

### 14.2 KPTI 비용 무시
DB / NIC heavy 워크로드 영향. PCID 활용 CPU.

### 14.3 user 코드의 kernel 가정
ring 0 명령 시도 → SIGSEGV.

### 14.4 vDSO 없는 라이브러리
직접 syscall → 느림. glibc 의 적절한 버전.

### 14.5 strace 의 overhead
syscall 마다 trace → 응용 매우 느림.

### 14.6 컴파일러 syscall inline
직접 호출보다 glibc wrapper 가 errno / signal 안전.

---

## 15. 학습 자료

- **The Linux Programming Interface** Ch. 3
- **Intel SDM Vol. 3** Ch. 6 (Interrupts) + Ch. 5 (System Architecture)
- **kernel.org** `Documentation/x86/...`
- **Brendan Gregg — KPTI 영향**

---

## 16. 관련

- [[syscall]] — Syscall hub
- [[posix-calls]]
- [[../scheduling/context-switch]]
