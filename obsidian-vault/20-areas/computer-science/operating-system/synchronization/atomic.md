---
title: "Atomic / CAS / Memory Order — Lock-free 기초"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:30:00+09:00
tags:
  - operating-system
  - sync
  - atomic
  - lock-free
---

# Atomic / CAS / Memory Order

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | atomic 연산 / CAS / Memory Order |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

CPU 가 보장하는 **원자 연산 + 메모리 순서**. 락 없이 동시성 가능 (lock-free).

---

## 2. 기본 연산

| 연산 | 의미 |
| --- | --- |
| `load(p)` | 원자 읽기 |
| `store(p, v)` | 원자 쓰기 |
| `exchange(p, v)` | 원자 교환 → 이전 값 |
| `fetch_add(p, n)` | += n + 이전 값 반환 |
| `fetch_sub`, `fetch_or`, `fetch_and`, `fetch_xor` | 동일 |
| **`compare_exchange(p, expected, desired)`** (CAS) | `*p == expected` 면 `desired` 로 교체, 결과 bool |
| `test_and_set` | 비트 설정 + 이전 |

---

## 3. CAS (Compare-And-Swap) — 핵심

```c
bool cas(int *p, int expected, int desired) {
    // atomic:
    if (*p == expected) { *p = desired; return true; }
    return false;
}
```

거의 모든 lock-free 알고리즘의 기반.

### 3.1 ABA 문제

```
스레드 1: 본 값 = A
스레드 2: A → B → A (다른 값으로 됐다 돌아옴)
스레드 1: CAS(A, X) 성공 — but 사이에 변화 있었음
```

해결:
- 버전 카운터 / DCAS (double-word CAS)
- Hazard pointer
- RCU / Epoch-based reclamation

---

## 4. C++ std::atomic

```cpp
#include <atomic>
std::atomic<int> counter{0};

counter.fetch_add(1);              // ++ 안전
int v = counter.load();
counter.store(10);

int expected = 5;
counter.compare_exchange_weak(expected, 10);
//  *counter == expected ? *counter = 10 : expected = *counter
```

### 4.1 weak vs strong

```cpp
compare_exchange_weak    // spurious fail 가능 — loop 안에서 OK
compare_exchange_strong  // spurious fail X — 단발 사용
```

```cpp
int old = atom.load();
while (!atom.compare_exchange_weak(old, old + 1))
    ;   // loop
```

---

## 5. Memory Order

CPU 가 명령을 reorder → atomic 만으로 동기화 안 될 수 있음.

```cpp
std::memory_order_relaxed     // 순서 보장 X — 카운터에 OK
std::memory_order_acquire     // 이 이후의 load/store 가 앞으로 못 옴
std::memory_order_release     // 이 이전의 load/store 가 뒤로 못 감
std::memory_order_acq_rel     // = acquire + release (RMW 용)
std::memory_order_seq_cst     // 가장 강함 (기본) — 전 코어 동일 순서
```

### 5.1 Release / Acquire 페어

```cpp
// 생산자
data = 42;
flag.store(1, std::memory_order_release);

// 소비자
while (flag.load(std::memory_order_acquire) == 0) ;
assert(data == 42);  // 보장
```

→ release 이전의 모든 write 가 acquire 이후의 read 에서 visible.

---

## 6. C++ atomic_flag

```cpp
std::atomic_flag f = ATOMIC_FLAG_INIT;
while (f.test_and_set(std::memory_order_acquire)) ;
// critical
f.clear(std::memory_order_release);
```

가장 단순한 spinlock.

---

## 7. C11

```c
#include <stdatomic.h>
atomic_int counter = 0;
atomic_fetch_add(&counter, 1);
atomic_compare_exchange_strong(&p, &expected, desired);
```

---

## 8. GCC builtins

```c
int v = __atomic_load_n(&x, __ATOMIC_ACQUIRE);
__atomic_store_n(&x, 5, __ATOMIC_RELEASE);
__atomic_fetch_add(&x, 1, __ATOMIC_RELAXED);
__atomic_compare_exchange_n(&x, &exp, des, weak, succ, fail);
```

---

## 9. Java

```java
AtomicInteger c = new AtomicInteger(0);
c.incrementAndGet();
c.compareAndSet(expected, new_value);

// volatile
volatile boolean ready;
```

`volatile` = sequentially consistent on JVM 5+ (happens-before 보장).

`VarHandle` (9+) — 더 정밀한 memory order.

---

## 10. Rust

```rust
use std::sync::atomic::{AtomicI32, Ordering};

let c = AtomicI32::new(0);
c.fetch_add(1, Ordering::Relaxed);
c.compare_exchange(1, 2, Ordering::AcqRel, Ordering::Acquire);
```

memory order 명시 강제 → 더 안전.

---

## 11. Go

```go
import "sync/atomic"

var c atomic.Int64
c.Add(1)
c.CompareAndSwap(1, 2)
c.Load()
c.Store(0)
```

Go memory model: happens-before via channels / atomic / sync.Mutex.

---

## 12. False Sharing

```c
struct {
    atomic_int a;     // core 1
    atomic_int b;     // core 2
} pad;
```

a 와 b 가 같은 cache line (64B) → 한 코어가 a 수정 시 다른 코어의 b 캐시 invalidate.

해결:
```c
struct {
    atomic_int a;
    char _pad[60];
    atomic_int b;
} __attribute__((aligned(64)));
```

C++17: `alignas(std::hardware_destructive_interference_size)`.

→ 성능에 큰 영향. perf 의 cache-misses 로 측정.

---

## 13. Lock-Free Queue / Stack — 어려움

```cpp
// Treiber Stack — 단순 push/pop
struct Node { Node* next; T data; };
std::atomic<Node*> top;

void push(T x) {
    Node* n = new Node{nullptr, x};
    Node* t;
    do {
        t = top.load();
        n->next = t;
    } while (!top.compare_exchange_weak(t, n));
}

T pop() {
    Node* t;
    do {
        t = top.load();
        if (!t) return T{};
    } while (!top.compare_exchange_weak(t, t->next));
    T x = t->data; delete t; return x;     // delete 시 ABA + UAF!
}
```

→ delete 가 어렵다. Hazard Pointer / Epoch / RCU 필요.

→ 실무: **잘 검증된 라이브러리** (folly, concurrentqueue 등) 사용.

---

## 14. Lock-Free vs Wait-Free

| 종류 | 의미 |
| --- | --- |
| Obstruction-free | 다른 스레드 없으면 진행 |
| Lock-free | 어떤 스레드는 진행 |
| Wait-free | 모든 스레드가 유한 단계에 완료 |

대부분의 실용 라이브러리 = lock-free. Wait-free 는 거의 학술.

---

## 15. 함정

### 15.1 `volatile` 만으로 atomic
**C/C++ 의 volatile 은 atomic X**. atomic 사용. (Java 의 volatile 은 다름 — happens-before.)

### 15.2 단어 크기 가정
4 / 8 byte 정렬된 word 만 atomic. 64-bit 값을 32-bit 머신에서?

### 15.3 memory_order_relaxed 남용
순서 보장 X — bug 의 원인.

### 15.4 ABA
CAS 만으로 부족. version counter / hazard pointer.

### 15.5 False sharing
hot atomic 변수 cache line 분리.

### 15.6 SeqCst 만 쓰기
모든 atomic 을 SeqCst → 느림 (fence 폭증). 필요할 때만.

### 15.7 atomic 의 합성 안전 X
atomic op 2 개 = atomic 아님. 단일 op 단위.

### 15.8 RMW 의 contention
hot counter 가 코어 수십 개에서 fetch_add → cache line bouncing. counter 분할.

---

## 16. 학습 자료

- **C++ Concurrency in Action** Ch. 5
- **The Art of Multiprocessor Programming** Ch. 5-7
- **Memory Models** — preshing.com
- **Linux Memory Barriers** — Documentation/memory-barriers.txt
- **Jeff Preshing 블로그** — 가장 좋은 atomic 입문

---

## 17. 관련

- [[mutex]] — 락 기반 대안
- [[spinlock]] — atomic 응용
- [[synchronization]] — Sync hub
