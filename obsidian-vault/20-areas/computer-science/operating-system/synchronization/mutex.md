---
title: "Mutex — 상호 배제"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:05:00+09:00
tags:
  - operating-system
  - sync
  - mutex
---

# Mutex

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Mutex / Recursive / Adaptive / Linux futex |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

**Mut**ual **Ex**clusion — 한 번에 하나의 스레드만 임계영역 진입.

---

## 2. POSIX

```c
#include <pthread.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&m);
// critical section
pthread_mutex_unlock(&m);

pthread_mutex_trylock(&m);          // 즉시 실패 가능
pthread_mutex_timedlock(&m, &ts);   // 타임아웃

pthread_mutex_destroy(&m);
```

### 2.1 attr

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&m, &attr);
```

| Type | 의미 |
| --- | --- |
| `NORMAL` (기본) | 같은 스레드가 두번 lock → deadlock |
| `RECURSIVE` | 같은 스레드 N 번 OK (N 번 unlock) |
| `ERRORCHECK` | 잘못된 사용에 에러 반환 |
| `DEFAULT` | 구현 의존 |

---

## 3. 언어별

### 3.1 C++

```cpp
#include <mutex>
std::mutex m;

{
    std::lock_guard<std::mutex> lock(m);    // RAII
    // critical
}   // 자동 unlock

std::unique_lock<std::mutex> l(m, std::defer_lock);
l.lock();
// ...
l.unlock();

std::recursive_mutex rm;
std::timed_mutex tm;
```

### 3.2 Java

```java
synchronized (lock) {
    // critical
}

ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // critical
} finally {
    lock.unlock();
}

// tryLock + timeout
if (lock.tryLock(1, TimeUnit.SECONDS)) { ... }
```

### 3.3 Python

```python
import threading
lock = threading.Lock()

with lock:
    # critical
    ...

# RLock = recursive
rlock = threading.RLock()
```

### 3.4 Rust

```rust
use std::sync::Mutex;

let m = Mutex::new(0);
{
    let mut guard = m.lock().unwrap();
    *guard += 1;
}   // Drop = 자동 unlock
```

### 3.5 Go

```go
var mu sync.Mutex

mu.Lock()
defer mu.Unlock()
// critical
```

---

## 4. Mutex 의 구현 — futex (Linux)

```
uncontended (경합 X):
  atomic CAS 만 — 사용자 공간, 매우 빠름 (~10 ns)

contended (경합 O):
  futex syscall → 커널이 wait queue 에 보관
```

→ 평소엔 가벼움. 경합 시만 syscall.

### 4.1 futex op

```c
syscall(SYS_futex, &word, FUTEX_WAIT, expected, timeout);
syscall(SYS_futex, &word, FUTEX_WAKE, 1);
```

→ pthread_mutex / glibc 가 사용.

---

## 5. Recursive Mutex

같은 스레드가 여러 번 lock 가능. 같은 횟수 unlock 필요.

```c
pthread_mutex_t m;   // RECURSIVE
pthread_mutex_lock(&m);
pthread_mutex_lock(&m);   // OK (count = 2)
pthread_mutex_unlock(&m);
pthread_mutex_unlock(&m); // 해제
```

⚠️ "재귀 락이 필요하면 디자인 결함" 이라는 시각도 (Linus / Joe Duffy). 신중히.

---

## 6. Adaptive Mutex

```
1. spin 짧게 (수십 cycle)
2. 그래도 안 풀리면 → 진짜 sleep (futex)
```

짧은 critical section + 멀티코어에 효율적. Solaris, JVM, glibc 일부 활성.

---

## 7. Priority Inversion

```
Low priority thread L 이 mutex 보유
High priority thread H 가 mutex 대기
Medium thread M 이 L 을 preempt → H 는 기다림
```

해결:
- **Priority Inheritance** — L 의 우선순위를 H 까지 올림
- **Priority Ceiling** — mutex 자체에 ceiling 우선순위

```c
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
```

실시간 시스템 (RTOS) 의 중요 함정. 화성 탐사선 Pathfinder 사고 (1997).

---

## 8. Try / Timed

```c
if (pthread_mutex_trylock(&m) == 0) {
    // got it
    pthread_mutex_unlock(&m);
} else {
    // 다른 처리
}

struct timespec ts;
clock_gettime(CLOCK_REALTIME, &ts);
ts.tv_sec += 5;
if (pthread_mutex_timedlock(&m, &ts) == 0) { ... }
```

→ deadlock 회피 / SLO 보장.

---

## 9. Mutex vs Spinlock

| | Mutex | Spinlock |
| --- | --- | --- |
| 대기 | sleep (futex) | busy wait |
| 비용 | context switch | CPU 점유 |
| critical section | 길어도 OK | 짧아야 함 |
| 유저 / 커널 | 둘 다 | 주로 커널 |
| 멀티코어 | 효율 | 효율 (짧을 때) |

자세히 → [[spinlock]]

---

## 10. 적용 패턴

### 10.1 Coarse-grained
```c
mutex_lock(&big_lock);
// 전체 작업
mutex_unlock(&big_lock);
```

단순 / 경합 ↑.

### 10.2 Fine-grained
```c
mutex_lock(&row->lock);
// 특정 row 만
mutex_unlock(&row->lock);
```

복잡 / 경합 ↓ / deadlock 위험 ↑.

### 10.3 Lock-free
mutex 안 씀 — atomic / CAS.
자세히 → [[atomic]]

---

## 11. Distributed Mutex

여러 노드 사이의 mutex — Redis SET NX EX / ZooKeeper / etcd.
자세히 → [[../../database/redis/distributed-lock]]

---

## 12. 함정

### 12.1 Lock 안에서 외부 호출 / callback
재진입 / deadlock 위험.

### 12.2 RAII 없는 unlock 누락
예외 / return path. C++ lock_guard / try-finally / Rust Drop.

### 12.3 큰 critical section
경합 폭증. 짧게.

### 12.4 Deadlock — 락 순서
[[deadlock]] 참조.

### 12.5 Recursive 의존
디자인 검토 — flat 구조가 더 안전.

### 12.6 Mutex + fork
자식이 락 상태 그대로 — 자식이 그 락에 접근하면 deadlock. fork 후 즉시 exec.

### 12.7 Mutex 가 stack 변수
스코프 끝나면 destroy. 다른 스레드 사용 중이면 UB.

### 12.8 NORMAL mutex 의 double-lock
같은 스레드가 두 번 → deadlock. ERRORCHECK 로 디버그.

---

## 13. 학습 자료

- **The Art of Multiprocessor Programming** Ch. 2-7
- `man 3 pthread_mutex_lock`
- **Mutex Documentation** — Linux kernel doc

---

## 14. 관련

- [[semaphore]] — counting
- [[condvar]] — 조건 대기
- [[spinlock]] — busy wait
- [[deadlock]]
- [[atomic]] — lock-free 대안
- [[synchronization]] — Sync hub
