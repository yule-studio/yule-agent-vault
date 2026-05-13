---
title: "동기화 (Synchronization) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:00:00+09:00
tags:
  - operating-system
  - sync
  - hub
---

# 동기화 — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 8 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 왜 동기화

여러 스레드 / 프로세스가 같은 자원 (메모리, 파일, DB) 을 동시에 다룸 → **race condition** + **inconsistency**.

```
스레드 A: x = x + 1    ; (load, +1, store)
스레드 B: x = x + 1
→ 결과: 1, 2 둘 다 가능
```

---

## 2. 임계 구역 (Critical Section)

여러 스레드가 동시 진입하면 안 되는 코드. 상호 배제 (Mutual Exclusion) 보장 필요.

```
entry section   ← 락 획득
  critical section
exit section    ← 락 해제
```

### 2.1 좋은 동기화의 4 조건

1. **Mutual Exclusion** — 한 번에 하나
2. **Progress** — 누군가는 진입 가능 (deadlock 없음)
3. **Bounded Waiting** — 무한 대기 없음 (starvation 없음)
4. **No assumption** — 속도 / 코어 수에 무관

---

## 3. 도구 한눈에

| 도구 | 용도 | 노트 |
| --- | --- | --- |
| **Mutex** | 1 스레드 진입 | [[mutex]] |
| **Semaphore** | N 허용 (counting) / 1 (binary) | [[semaphore]] |
| **Condition Variable** | 조건 만족 대기 | [[condvar]] |
| **Read-Write Lock** | reader 다수 / writer 단독 | [[rwlock]] |
| **Spinlock** | busy-wait, 짧은 critical section | [[spinlock]] |
| **Atomic / CAS** | lock-free, 단일 변수 | [[atomic]] |
| **Monitor** | 객체 + 자동 락 (Java synchronized) | (Java) |
| **Barrier** | 모든 스레드 도달까지 대기 | (POSIX) |
| **Latch / CountDownLatch** | 1회용 barrier | (Java) |
| **RCU** | Read-Copy-Update (커널) | (Linux) |

---

## 4. Deadlock

자세히 → [[deadlock]]

Coffman 4 조건 모두 만족 시 발생:
1. Mutual Exclusion
2. Hold and Wait
3. No Preemption
4. Circular Wait

해결: 락 순서 고정 / try_lock / timeout.

---

## 5. 고전 동기화 문제

자세히 → [[classic-problems]]

- Producer-Consumer
- Readers-Writers
- Dining Philosophers
- Sleeping Barber
- Cigarette Smokers

---

## 6. 메모리 모델

CPU 가 명령을 reorder 할 수 있음 → 동기화 의도 깨질 수 있음.

```
- x86 — relatively strong (TSO)
- ARM / POWER — weak — explicit barrier 필요
```

해결:
- `volatile` (Java) — visibility + happens-before
- `std::atomic` (C++) — memory_order
- `pthread_mutex` 가 자동 fence
- `mfence` / `dmb` 같은 명령

---

## 7. lock-free / wait-free

| 종류 | 의미 |
| --- | --- |
| Blocking | 락 사용 |
| Lock-free | 어떤 스레드가 끝나면 다른 스레드는 진행 가능 |
| Wait-free | 모든 스레드가 유한 단계에 끝남 |

CAS / atomic 기반. 어렵지만 contention 적음 / latency 일정.

자세히 → [[atomic]]

---

## 8. Linux 의 futex

```
futex (Fast Userspace muTEX)
contention 없으면 syscall 없이 atomic 만
contention 시 syscall 로 wait
→ pthread_mutex, semaphore 의 기반
```

---

## 9. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[mutex]] | Mutex / Recursive / Adaptive |
| [[semaphore]] | Counting / Binary |
| [[condvar]] | wait / signal / broadcast / spurious |
| [[rwlock]] | shared / exclusive / starvation |
| [[spinlock]] | busy-wait 언제 |
| [[atomic]] | CAS, fetch_add, memory order |
| [[deadlock]] | 4 조건 / 회피 / 검출 |
| [[classic-problems]] | producer-consumer 등 고전 |

---

## 10. 함정 (요약)

### 함정 1 — 락 순서 불일치 → deadlock
A→B, B→A. 항상 같은 순서.

### 함정 2 — Double-Checked Locking
`volatile` 없으면 깨짐 (Java 1.4 이전).

### 함정 3 — busy-wait
`while (!ready) {}` → CPU 100%. condvar / sleep / epoll.

### 함정 4 — Lock 안에서 외부 호출 (callback)
재진입 / deadlock 가능.

### 함정 5 — Mutex + fork
fork 시 락 상태 보존. async-fork-safe X.

### 함정 6 — RAII 없는 lock/unlock
예외 시 unlock 누락. C++ lock_guard / Rust Drop / Java try-finally.

### 함정 7 — `volatile` ≠ atomic (C/C++)
`volatile` 은 reorder 방지일 뿐, race 막지 X.

---

## 11. 학습 자료

- **The Art of Multiprocessor Programming** — Herlihy / Shavit (필독)
- **Java Concurrency in Practice** — Brian Goetz
- **C++ Concurrency in Action** — Anthony Williams
- **Is Parallel Programming Hard?** — Paul McKenney (Linux 커널)
- **Linux Kernel Synchronization** — Documentation/locking/

---

## 12. 관련

- [[../threads/threads]] — 스레드 + 락
- [[../scheduling/scheduling]] — 락 + 대기
- [[../operating-system|↑ OS hub]]
