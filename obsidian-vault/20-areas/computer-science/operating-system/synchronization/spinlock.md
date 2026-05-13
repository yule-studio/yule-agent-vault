---
title: "Spinlock — busy-wait 락"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:25:00+09:00
tags:
  - operating-system
  - sync
  - spinlock
---

# Spinlock

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | busy-wait / 적용 / 변형 |

**[[synchronization|↑ Sync hub]]**

</br>

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

```
while (!try_lock()) ;   // spin (busy wait)
critical section
unlock();
```

sleep 안 함 — CPU 점유한 채 락 풀릴 때까지 회전.

---

## 2. 언제 쓰나

- **critical section 매우 짧음** (수십 ns ~ μs)
- context switch 비용 > spin 비용
- 멀티코어 (다른 코어가 곧 풀어줌)

전형: **OS 커널 내부**.

---

## 3. 안 쓰는 경우

- 단일 코어 (다른 작업 못 함 — futex / mutex)
- 긴 critical section
- 우선순위 다른 스레드 사이 (priority inversion 심각)
- 일반 userspace 의 락 — 보통 mutex 가 정답

---

## 4. POSIX

```c
pthread_spinlock_t s;
pthread_spin_init(&s, PTHREAD_PROCESS_PRIVATE);
pthread_spin_lock(&s);
// critical
pthread_spin_unlock(&s);
pthread_spin_destroy(&s);
```

⚠️ Linux 의 `pthread_spinlock_t` 는 단순 busy-wait. 우선순위 / 스케줄링 / Preemption 등 위험.

---

## 5. 간단 구현

### 5.1 Test-and-Set

```c
typedef volatile int spinlock_t;

void lock(spinlock_t *l) {
    while (__atomic_test_and_set(l, __ATOMIC_ACQUIRE))
        ;
}
void unlock(spinlock_t *l) {
    __atomic_clear(l, __ATOMIC_RELEASE);
}
```

→ 단순. cache line 경합 (모든 코어가 같은 line 에 write) → 성능 저하.

### 5.2 Test-and-Test-and-Set (TTAS)

```c
void lock(spinlock_t *l) {
    while (1) {
        while (*l) ;                              // 1단 read (shared cache 가능)
        if (!__atomic_test_and_set(l, ACQ)) break; // 2단 write (배타)
    }
}
```

→ 평소엔 cache shared read — 경합 ↓.

### 5.3 Ticket Lock

```c
typedef struct { int next, owner; } ticket_lock_t;

void lock(ticket_lock_t *l) {
    int my = __atomic_fetch_add(&l->next, 1, RELAXED);
    while (__atomic_load_n(&l->owner, ACQ) != my)
        ;
}
void unlock(ticket_lock_t *l) {
    __atomic_fetch_add(&l->owner, 1, RELEASE);
}
```

→ FIFO. fairness.

### 5.4 MCS Lock

각 스레드가 자기 노드에 spin (cache line 분리) — 대규모 멀티코어 (수십~수백) 에 표준.

---

## 6. PAUSE 명령

```c
while (locked) {
    __builtin_ia32_pause();   // x86 "PAUSE"
}
```

- spin 중 HT 의 다른 thread 에 양보
- 전력 ↓
- ARM: `__yield()`

---

## 7. Adaptive (mutex)

```
짧게 spin → 그래도 안 풀리면 sleep (futex)
```

대부분 현대 mutex (glibc, JVM) 가 적응적.

---

## 8. RW Spinlock

```c
typedef struct { int readers; } rwspinlock_t;

void read_lock(rwspinlock_t *l) {
    while (1) {
        int r = __atomic_load_n(&l->readers, ACQ);
        if (r >= 0 && __atomic_compare_exchange_n(&l->readers, &r, r+1, ...)) break;
    }
}
void write_lock(rwspinlock_t *l) {
    while (1) {
        int r = 0;
        if (__atomic_compare_exchange_n(&l->readers, &r, -1, ...)) break;
    }
}
```

→ 커널 안의 rwlock 도 종종 spinlock 기반.

---

## 9. 단일 코어의 위험

```
Single core:
  Thread A spinlock 보유
  Thread B 가 spinlock 시도 → spin
  → A 가 깨어나려면 B 의 timeslice 끝나야 함
  → CPU 100% + 전체 정지
```

→ 단일 코어 / VM 1 vCPU 에선 spinlock 사용 X.

---

## 10. 우선순위 (Priority Inversion)

Spinlock 은 owner 정보 X → priority inheritance 어려움. 실시간 OS 에선 위험.

→ 보통 RTOS 는 PIP (Priority Inheritance Protocol) mutex 사용.

---

## 11. Linux 커널의 spinlock

```c
spinlock_t lock;
spin_lock_init(&lock);
spin_lock(&lock);          // 인터럽트 컨텍스트 X
spin_lock_irqsave(&lock, flags);  // 인터럽트 disable
// critical
spin_unlock_irqrestore(&lock, flags);
```

커널 코드는 sleep 불가 컨텍스트 (인터럽트 핸들러) 가 많음 → mutex 대신 spinlock.

---

## 12. 측정

```bash
# x86 perf
perf stat -e cycles,instructions,cache-misses,LLC-load-misses ./bench
```

spin 의 cache miss / lock contention 측정.

---

## 13. 함정

### 13.1 userspace 에서 spinlock 남용
대부분 mutex 가 정답. spin 은 micro-optimization.

### 13.2 단일 코어
완전 정지.

### 13.3 인터럽트 핸들러 + spinlock
인터럽트 disable 안 하면 deadlock 가능 — `spin_lock_irqsave`.

### 13.4 PAUSE 안 씀
경합 ↑, 전력 ↑.

### 13.5 긴 critical section
다른 모든 코어 정지.

### 13.6 unfair spin
ticket / MCS 로 starvation 해결.

### 13.7 NUMA + spinlock
cache line bouncing 폭증. NUMA-aware 락.

---

## 14. 학습 자료

- **The Art of Multiprocessor Programming** Ch. 7
- **Is Parallel Programming Hard?** — Paul McKenney
- Linux Kernel `Documentation/locking/spinlocks.rst`

---

## 15. 관련

- [[mutex]] — sleep 기반
- [[atomic]] — CAS / TAS
- [[synchronization]] — Sync hub
