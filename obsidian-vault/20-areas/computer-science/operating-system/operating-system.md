---
title: "운영체제 (Operating System)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T10:30:00+09:00
tags:
  - operating-system
  - process
  - thread
  - memory
  - filesystem
---

# 운영체제 (Operating System)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**하드웨어와 응용 프로그램 사이의 관리자**. 자원 (CPU/메모리/디스크/I/O) 을
프로세스에게 추상화 + 분배 + 보호.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1960s | OS/360 (IBM 메인프레임) — 배치 처리 |
| 1969 | UNIX (Ken Thompson, Dennis Ritchie, Bell Labs) |
| 1973 | UNIX C 로 재작성 — 이식성 혁명 |
| 1981 | MS-DOS |
| 1984 | Mac OS (Lisa OS 기반) |
| 1991 | Linux 0.01 (Linus Torvalds) |
| 1993 | Windows NT (Dave Cutler) |
| 2001 | macOS X (NeXTSTEP / Mach 기반) |
| 2007 | iOS / Android (Linux 기반) |

핵심 통찰: **추상화 + 보호 + 자원 다중화 (multiplexing)**.

---

## 3. 운영체제의 4 가지 역할

1. **프로세스 관리** — CPU 스케줄링, IPC
2. **메모리 관리** — 가상 메모리, 페이징
3. **파일 시스템** — 영속 저장
4. **I/O 관리** — 디바이스 드라이버

추가: 네트워크 스택, 보안, 사용자 관리.

---

## 4. 프로세스 vs 스레드

### 4.1 프로세스 (Process)

- 실행 중인 프로그램
- 독립된 주소 공간 (PCB: Process Control Block)
- 컨텍스트 스위치 비용 큼 (TLB flush, 캐시 무효화)

### 4.2 스레드 (Thread)

- 프로세스 내 실행 단위
- 주소 공간 공유 (heap, code)
- 자기만의 stack + register + PC

### 4.3 비교

| 기준 | 프로세스 | 스레드 |
| --- | --- | --- |
| 메모리 | 독립 | 공유 |
| 생성 비용 | 큼 | 작음 |
| 통신 | IPC (pipe, shm, socket) | 메모리 직접 |
| 안정성 | 크래시 격리 | 한 스레드 죽으면 전체 |
| 컨텍스트 스위치 | 무거움 | 가벼움 |

### 4.4 프로세스 상태

```
NEW → READY ⇄ RUNNING → TERMINATED
        ↑       ↓
        └ WAITING ←
```

### 4.5 fork / exec (UNIX)

```c
pid_t pid = fork();        // 자식 프로세스 복제
if (pid == 0) {
    execvp("ls", args);    // 새 프로그램으로 교체
} else {
    waitpid(pid, &status, 0);  // 자식 종료 대기
}
```

Copy-on-Write 로 fork 가 가벼움.

---

## 5. CPU 스케줄링

### 5.1 알고리즘

| 알고리즘 | 설명 | 장단점 |
| --- | --- | --- |
| FCFS | 도착 순 | 단순 / Convoy effect |
| SJF | 짧은 작업 우선 | 평균 대기 최소 / 기아 |
| **RR** (Round Robin) | 타임 슬라이스 | 공평 / quantum 선택 중요 |
| **MLFQ** | 다단계 피드백 | 적응적 / Linux/Solaris |
| **CFS** | Completely Fair Scheduler | Linux 2.6.23+ Red-Black Tree, vruntime |
| Priority | 우선순위 | 기아 → Aging |
| EDF | Earliest Deadline First | 실시간 |

### 5.2 Linux CFS (Completely Fair Scheduler)

- Red-Black Tree 로 task 의 vruntime 정렬
- 가장 작은 vruntime 의 task 실행
- nice 값으로 가중치 조정
- O(log N) 선택

### 5.3 컨텍스트 스위치 비용

- 약 1-10 마이크로초
- TLB / L1 캐시 무효화
- 빈번한 스위치 = thrashing

---

## 6. 동기화 (Synchronization)

### 6.1 Race Condition

```
스레드 A: x = x + 1
스레드 B: x = x + 1

기계어:
  load x → reg
  add 1
  store reg → x

인터리빙으로 결과가 1 일 수도, 2 일 수도.
```

### 6.2 Critical Section

상호 배제 (Mutual Exclusion) 보장 필요.

### 6.3 동기화 도구

| 도구 | 설명 |
| --- | --- |
| **Mutex** | 1 개 스레드만 진입 |
| **Semaphore** | N 개 허용 (counting), 1 개 (binary) |
| **Condition Variable** | 조건 만족 대기 |
| **Read-Write Lock** | reader 다수 / writer 단독 |
| **Spinlock** | busy-wait, 짧은 임계영역 |
| **Atomic** | CAS, fetch_add 등 lock-free |
| **Monitor** | 객체 + 자동 락 (Java synchronized) |
| **Barrier** | 모든 스레드 도달까지 대기 |

### 6.4 데드락 (Deadlock)

4 가지 조건 (Coffman 1971) 모두 만족 시:
1. 상호 배제
2. 점유 + 대기 (Hold and Wait)
3. 비선점 (No Preemption)
4. 순환 대기 (Circular Wait)

**예방** — 4 조건 중 하나 깨기 (예: 락 순서 고정).
**회피** — Banker's Algorithm.
**탐지 + 복구** — 자원 그래프.
**무시** — Ostrich algorithm (Linux, Windows).

### 6.5 고전 문제

- **Producer-Consumer** — 버퍼 + condvar
- **Readers-Writers** — RW lock
- **Dining Philosophers** — 락 순서 / monitor
- **Sleeping Barber** — semaphore

---

## 7. 메모리 관리

### 7.1 메모리 계층 (재확인)

```
Register     <  1 ns
L1 cache     ~  1 ns
L2 cache     ~  4 ns
L3 cache     ~ 10 ns
RAM          ~ 100 ns
NVMe SSD     ~ 100 µs
HDD          ~ 10 ms
Network      ~ 100 ms
```

### 7.2 가상 메모리

- 각 프로세스는 자기만의 0 ~ 2^64 가상 주소 공간 환상
- MMU + 페이지 테이블이 물리 주소 변환
- **페이지** (4KB) 단위 — Huge Page 2MB / 1GB

### 7.3 페이지 테이블

```
가상 주소: [페이지 번호 | 오프셋]
                ↓
           페이지 테이블 → 물리 페이지 프레임
                ↓
물리 주소: [프레임 번호 | 오프셋]
```

- 다단계 (4-level on x86-64)
- TLB (Translation Lookaside Buffer) 캐시

### 7.4 페이지 폴트

- 페이지가 RAM 에 없음 → OS 가 디스크에서 로드
- Major fault — 디스크 I/O 필요
- Minor fault — RAM 에는 있지만 매핑 안 됨

### 7.5 페이지 교체 알고리즘

| 알고리즘 | 설명 |
| --- | --- |
| FIFO | 단순 / Belady's anomaly |
| **LRU** | 최근 사용 안 한 것 | Hash + DLL |
| **Clock** | LRU 근사 | Use bit |
| **LFU** | 사용 횟수 최소 |
| **Working Set** | 최근 시간 윈도우 |
| **2Q / ARC** | LRU + Frequency |

### 7.6 단편화

- **외부 단편화** — 빈 공간 분산. Buddy / Slab 할당자.
- **내부 단편화** — 할당 단위 > 요청. 페이지 마지막 부분.

### 7.7 메모리 할당자

- **Buddy System** — 2^k 단위 분할 / 병합. Linux 페이지 할당.
- **Slab Allocator** — 자주 쓰는 객체 풀. Linux 커널.
- **jemalloc / tcmalloc** — userspace 멀티스레드 친화.

### 7.8 mmap

- 파일 / 익명 메모리를 가상 주소 공간에 매핑
- 큰 파일을 부분 접근 — DB / 검색 엔진

---

## 8. 파일 시스템

### 8.1 추상화

- **inode** — 파일 메타데이터 + 데이터 블록 포인터
- **dentry** — 디렉토리 엔트리 (이름 → inode)
- **File Descriptor** — 프로세스 별 파일 핸들

### 8.2 주요 파일 시스템

| FS | 출시 | 특징 |
| --- | --- | --- |
| ext2 → ext4 | Linux 기본 | journaling, extents |
| **XFS** | 고성능 / 큰 파일 |
| **Btrfs / ZFS** | snapshot, COW, RAID |
| **APFS** | macOS, 2017 |
| **NTFS** | Windows |
| **FAT32 / exFAT** | USB |
| **F2FS** | Flash 최적화 (Samsung) |

### 8.3 Journaling

- 변경 사항을 먼저 로그 → 디스크 적용 → 로그 삭제
- 크래시 시 복구 가능

### 8.4 Copy-on-Write

- 수정 시 새 블록 작성, 메타데이터만 갱신
- 스냅샷 / 롤백 쉬움 (ZFS, Btrfs, APFS)

### 8.5 RAID

- **RAID 0** — Stripe (속도, 안정성 없음)
- **RAID 1** — Mirror (안정성)
- **RAID 5** — Stripe + parity (1 디스크 실패 OK)
- **RAID 6** — Stripe + 2 parity (2 디스크 OK)
- **RAID 10** — Mirror + Stripe

---

## 9. I/O 관리

### 9.1 I/O 모델

| 모델 | 설명 | 예 |
| --- | --- | --- |
| **Blocking** | 호출자 대기 | `read()` 기본 |
| **Non-blocking** | 즉시 반환 | `O_NONBLOCK` |
| **I/O Multiplexing** | 여러 FD 모니터 | select/poll/epoll/kqueue |
| **Signal-driven** | SIGIO | 드물게 |
| **Asynchronous I/O** | 완료 시 callback | io_uring / IOCP |

### 9.2 epoll (Linux) / kqueue (BSD/macOS) / IOCP (Windows)

- 수천 ~ 수만 연결 처리 (C10K 문제 해결)
- Nginx, Redis, Node.js 의 기반

### 9.3 io_uring (Linux 5.1+)

- 진정한 비동기, 시스템 콜 오버헤드 최소
- ring buffer 공유

### 9.4 Zero-Copy

- `sendfile()` — 디스크 → 소켓 직접
- 일반: read → user buffer → write (4 회 복사)
- Zero-copy: 1-2 회 (DMA)

---

## 10. 시스템 콜 (System Call)

### 10.1 User mode vs Kernel mode

- **Ring 3 (User)** — 응용 프로그램
- **Ring 0 (Kernel)** — 특권 명령
- `syscall` 명령 (x86-64) / `svc` (ARM) 으로 전환

### 10.2 주요 시스템 콜 (POSIX)

| 카테고리 | 호출 |
| --- | --- |
| 프로세스 | `fork`, `exec`, `wait`, `exit`, `kill` |
| 파일 | `open`, `read`, `write`, `close`, `lseek`, `stat` |
| 메모리 | `mmap`, `munmap`, `brk` |
| 네트워크 | `socket`, `bind`, `listen`, `accept`, `connect` |
| IPC | `pipe`, `shmget`, `msgget`, `semop` |
| 시그널 | `signal`, `sigaction`, `kill` |
| 시간 | `gettimeofday`, `clock_gettime` |

### 10.3 strace

```bash
strace -e trace=openat,read,write ./program
```

시스템 콜 디버깅 필수.

---

## 11. IPC (Inter-Process Communication)

| 방법 | 설명 |
| --- | --- |
| **Pipe** | 부모-자식 단방향 |
| **FIFO (Named pipe)** | 무관한 프로세스 간 |
| **Message Queue** | 메시지 큐 (System V / POSIX) |
| **Shared Memory** | 가장 빠름, 동기화 직접 |
| **Semaphore** | 동기화 |
| **Socket** | 같은 / 다른 호스트 |
| **Signal** | 비동기 알림 (SIGTERM, SIGKILL) |

---

## 12. 함정 / 안티패턴

### 함정 1 — 락 순서 불일치
A→B, B→A 데드락. 락 순서 고정.

### 함정 2 — Double-Checked Locking
```java
if (instance == null) {
    synchronized(this) {
        if (instance == null) instance = new X();
    }
}
```
volatile 없으면 동작 안 함 (Java 1.4 이전 깨짐).

### 함정 3 — busy-wait
```python
while not ready: pass    # CPU 100%
```
condvar / sleep / epoll 사용.

### 함정 4 — fork() 후 멀티스레드
fork 는 호출 스레드만 복사. 다른 스레드의 락 = 영원히 데드락.

### 함정 5 — 페이지 폴트 무시
큰 메모리 할당 후 첫 접근 시 폴트 폭발. `mmap + MAP_POPULATE` / mlock.

### 함정 6 — fsync 빼먹기
`write()` 만으로는 디스크 미반영. crash 시 손실.

### 함정 7 — TIME_WAIT 누적
SO_REUSEADDR, connection pool.

### 함정 8 — Out of Memory Killer
Linux 가 임의 프로세스 SIGKILL. cgroups / oom_score_adj 조정.

---

## 13. 컨테이너 / 가상화

### 13.1 가상화 (VM)

- 하드웨어 추상화 — 게스트 OS 가 자기 커널
- KVM, VMware, VirtualBox, Hyper-V
- 오버헤드 큼, 보안 강함

### 13.2 컨테이너

- 커널 공유 + 격리
- **namespace** (PID, NET, MNT, UTS, IPC, USER, CGROUP)
- **cgroups** — CPU / 메모리 / I/O 제한
- Docker, containerd, Podman

### 13.3 비교

| 기준 | VM | 컨테이너 |
| --- | --- | --- |
| 격리 | 강 | 약 (커널 공유) |
| 부팅 | 분 | 초 |
| 크기 | GB | MB |
| 오버헤드 | 큼 | 작음 |
| 보안 | 강 | 약 |

---

## 14. 면접 질문 모음

1. **프로세스 vs 스레드**.
2. **컨텍스트 스위치 비용** — TLB / 캐시 / 레지스터.
3. **데드락 4 조건과 해결**.
4. **페이지 폴트** — major / minor.
5. **CPU 캐시 / 가상 메모리 / TLB**.
6. **fork vs exec** — Copy-on-Write.
7. **세마포어 vs 뮤텍스**.
8. **epoll / select 차이**.
9. **시스템 콜 비용** — user → kernel 전환.
10. **VM vs 컨테이너**.

---

## 15. 학습 자료

- **OSTEP (Operating Systems: Three Easy Pieces)** — 무료 https://pages.cs.wisc.edu/~remzi/OSTEP/
- **Modern Operating Systems** (Tanenbaum)
- **Linux Kernel Development** (Robert Love)
- **The Linux Programming Interface** (Michael Kerrisk) — 시스템 콜 바이블
- **Understanding the Linux Kernel** (Bovet / Cesati)

---

## 16. 관련

- [[../computer-architecture/computer-architecture]] — CPU, 캐시
- [[../network/network]] — Socket, TCP/IP
- [[../distributed-systems/distributed-systems]] — IPC 확장
- [[../security-theory/security-theory]] — 권한, 격리
- [[../computer-science|↑ computer-science]]
