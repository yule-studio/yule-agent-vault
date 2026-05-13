---
title: "운영체제 (Operating System) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T10:30:00+09:00
tags:
  - operating-system
  - hub
  - process
  - memory
  - filesystem
  - linux
---

# 운영체제 (Operating System) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub 재편 + 깊이 분리 (process / memory / fs / io / linux 등) |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 단일 노트 (큰 틀) |

**[[../computer-science|↑ computer-science]]**

> 이 노트는 **OS 전체의 hub**. 각 개념은 별도 폴더 / 노트로 깊이 다룬다.
> 단일 사전이었던 v.1.0.0 은 hub 로 재편되어 다른 곳으로 분산됨.

---

## 0. OS 가 다루는 영역

OS = **하드웨어와 응용 사이의 관리자**. 4 가지 추상화:
- **CPU** → 프로세스 / 스레드 + 스케줄링
- **메모리** → 가상 메모리 + 페이지 테이블
- **디스크** → 파일 시스템
- **I/O** → 디바이스 / 네트워크 / 그래픽

추가: 동기화 / IPC / 시스템 콜 / 보안 / 가상화 / 컨테이너.

핵심 통찰 — **추상화 + 보호 + 자원 다중화 (multiplexing)**.

UNIX 1969 → C 재작성 1973 → Linux 1991 → 컨테이너 2013 까지의 누적이 오늘의 OS 생태.

---

## 1. 프로세스 / 스레드 — 깊이

- [[process/process|↗ Process hub]] — 프로그램 실행 단위, PCB, fork/exec, 시그널
- [[threads/threads|↗ Threads hub]] — 경량 실행 단위, 유저 / 커널 스레드, 풀

---

## 2. 동기화 — 깊이

[[synchronization/synchronization|↗ 동기화 hub]] — Race condition, 락, 데드락, 고전 문제

| 도구 | 용도 |
| --- | --- |
| Mutex | 1 스레드 진입 |
| Semaphore | N 허용 |
| Condvar | 조건 대기 |
| Spinlock | busy-wait |
| Atomic / CAS | lock-free |

---

## 3. 스케줄링 — 깊이

[[scheduling/scheduling|↗ Scheduling hub]] — FCFS / SJF / RR / MLFQ / CFS / EDF / 컨텍스트 스위치

---

## 4. 메모리 관리 — 깊이

[[memory/memory|↗ Memory hub]] — 가상 메모리 / 페이지 테이블 / TLB / 페이지 폴트 / 할당자 / mmap / Huge Page / NUMA

---

## 5. 파일 시스템 — 깊이

[[filesystem/filesystem|↗ Filesystem hub]] — inode / dentry / journaling / COW / RAID / fsync

---

## 6. I/O 관리 — 깊이

[[io/io|↗ I/O hub]] — Blocking / Non-blocking / select-poll-epoll / io_uring / Zero-copy

---

## 7. 시스템 콜 — 깊이

[[syscall/syscall|↗ Syscall hub]] — User/Kernel mode, POSIX, strace

---

## 8. IPC — 깊이

[[ipc/ipc|↗ IPC hub]] — Pipe / FIFO / Shared Memory / Message Queue / Unix Domain Socket / Signal

---

## 9. 가상화 / 컨테이너 — 깊이

[[virtualization/virtualization|↗ Virtualization hub]] — VM / Hypervisor / KVM / Namespace / cgroups / Container

---

## 10. 보안 — 깊이

[[security/security|↗ OS Security hub]] — Permission / Capability / SELinux / AppArmor / seccomp / Sandbox

---

## 11. OS 별 실무

- [[linux/linux|↗ Linux hub]] — 커널 / 배포판 / systemd / shell / 네트워크 / 모니터링 / 튜닝
- [[macos/macos|↗ macOS hub]] — XNU / Mach / launchd / APFS / Homebrew
- [[windows/windows|↗ Windows hub]] — NT 커널 / PowerShell / WSL / NTFS

---

## 12. 도구 / 실습

[[tools/tools|↗ OS 도구 hub]] — strace / perf / eBPF / lsof / gdb / dtrace / ftrace

---

## 13. 토픽 / 면접

[[topics/topics|↗ 토픽 hub]] — C10K, OOM Killer, Thrashing, Cache locality, 컨텍스트 스위치 비용, Userland vs Kernel

---

## 14. 핵심 키워드 한 줄

| 키워드 | 의미 |
| --- | --- |
| Kernel | 특권 모드의 OS 핵심 |
| Process / Thread | 실행 단위 |
| MMU / TLB | 가상 → 물리 주소 변환 |
| Page | 4KB 메모리 단위 |
| Syscall | user → kernel 진입 |
| epoll / io_uring | I/O 멀티플렉싱 |
| ext4 / XFS / ZFS | 파일 시스템 |
| Namespace / cgroup | 컨테이너의 토대 |
| Capability | fine-grained 권한 |
| OOM Killer | 메모리 부족 시 프로세스 종료 |

---

## 15. 면접 핵심 질문

1. **프로세스 vs 스레드** + 컨텍스트 스위치 비용.
2. **데드락 4 조건** + 해결.
3. **가상 메모리 / 페이지 테이블 / TLB**.
4. **fork vs exec** + Copy-on-Write.
5. **select / poll / epoll** 차이 + C10K.
6. **Semaphore vs Mutex**.
7. **시스템 콜 비용** — user → kernel.
8. **CFS** (Linux) 동작.
9. **VM vs 컨테이너** — Namespace / cgroup.
10. **fsync 가 왜 필요한가**.

---

## 16. 학습 자료

- **OSTEP** (Operating Systems: Three Easy Pieces) — 무료, 필독 (pages.cs.wisc.edu/~remzi/OSTEP)
- **Modern Operating Systems** — Tanenbaum
- **Linux Kernel Development** — Robert Love
- **The Linux Programming Interface** — Michael Kerrisk (시스템 콜 바이블)
- **Understanding the Linux Kernel** — Bovet / Cesati
- **System Performance** — Brendan Gregg
- **brendangregg.com / man7.org/linux** — 최고 수준 블로그 / 매뉴얼

---

## 17. 관련

- [[../computer-architecture/computer-architecture]] — CPU / 캐시 / 메모리 계층
- [[../network/network]] — Socket / TCP-IP / epoll
- [[../distributed-systems/distributed-systems]] — IPC 의 확장
- [[../security-theory/security-theory]] — 권한 / 격리 / 인증
- [[../../database/database]] — fsync, WAL, mmap 의 OS 의존
- [[../computer-science|↑ computer-science]]
