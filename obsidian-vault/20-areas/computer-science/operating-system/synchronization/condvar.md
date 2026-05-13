---
title: "Condition Variable — 조건 변수"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:15:00+09:00
tags:
  - operating-system
  - sync
  - condvar
---

# Condition Variable

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | wait / signal / broadcast / spurious |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

스레드가 **특정 조건이 만족될 때까지** mutex 와 함께 대기 / 신호 받기.

```
"큐가 비어있으면 대기"
"버퍼가 차면 대기"
"shutdown 신호 때까지 대기"
```

---

## 2. 패턴

```c
pthread_mutex_lock(&m);
while (!condition)              // ⚠️ while — if 아님
    pthread_cond_wait(&cv, &m);
// condition 만족 → 임계 코드
pthread_mutex_unlock(&m);
```

```c
// 깨우는 쪽
pthread_mutex_lock(&m);
condition = true;
pthread_cond_signal(&cv);       // 1 명
// 또는 pthread_cond_broadcast(&cv);   // 모두
pthread_mutex_unlock(&m);
```

---

## 3. `cond_wait` 의 동작

```
1. mutex 해제
2. wait queue 에 자기 추가
3. 깨어날 때까지 sleep
4. 깨어남 → mutex 다시 획득 (자동)
5. return
```

atomic 동작 — 1+2 사이에 시그널 놓침 없음.

---

## 4. while vs if — Spurious Wakeup

```c
// ❌ 위험
if (!cond) cond_wait();

// ✅ 안전
while (!cond) cond_wait();
```

이유:
- **Spurious wakeup** — POSIX 가 인정 — 가짜 깨어남 가능
- **Stolen wakeup** — 다른 스레드가 먼저 처리 후 조건 다시 거짓
- 거의 비용 0 → 항상 while.

---

## 5. signal vs broadcast

| | signal | broadcast |
| --- | --- | --- |
| 깨움 | 1 명 (구현 의존) | 모두 |
| 사용처 | 한 자원 추가 시 | 조건 큰 변화 / shutdown |
| 비용 | 작음 | 큼 |

### 5.1 언제 broadcast
- 여러 스레드가 조건 만족 가능 (Readers-Writers)
- shutdown / cancel
- 조건이 "변했음" 의미만, 누가 잡을지 모름

### 5.2 의심스러우면 broadcast
정확성 우선. 성능은 측정 후.

---

## 6. POSIX

```c
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

pthread_cond_wait(&cv, &m);
pthread_cond_timedwait(&cv, &m, &ts);
pthread_cond_signal(&cv);
pthread_cond_broadcast(&cv);
pthread_cond_destroy(&cv);
```

---

## 7. 언어별

### 7.1 C++

```cpp
#include <condition_variable>
std::mutex m;
std::condition_variable cv;
bool ready = false;

std::unique_lock<std::mutex> lock(m);
cv.wait(lock, [] { return ready; });   // predicate — while 내장
```

```cpp
// signal
{
    std::lock_guard<std::mutex> lock(m);
    ready = true;
}
cv.notify_one();    // or notify_all()
```

### 7.2 Java

```java
synchronized (lock) {
    while (!ready) lock.wait();
    // ...
}

// signal
synchronized (lock) {
    ready = true;
    lock.notify();          // or notifyAll()
}
```

또는 `Condition` (with ReentrantLock):

```java
Lock lock = new ReentrantLock();
Condition cond = lock.newCondition();

lock.lock();
try {
    while (!ready) cond.await();
} finally { lock.unlock(); }

cond.signalAll();
```

### 7.3 Python

```python
import threading
cv = threading.Condition()

with cv:
    while not ready:
        cv.wait()
    # ...

# signal
with cv:
    ready = True
    cv.notify_all()
```

### 7.4 Rust

```rust
use std::sync::{Mutex, Condvar};
let pair = Arc::new((Mutex::new(false), Condvar::new()));

let (lock, cvar) = &*pair;
let mut started = lock.lock().unwrap();
while !*started {
    started = cvar.wait(started).unwrap();
}
```

### 7.5 Go

Go 는 condvar 잘 안 씀 — channel 이 표준:

```go
done := make(chan struct{})
// wait
<-done
// signal
close(done)   // broadcast
```

또는 `sync.Cond`:

```go
var mu sync.Mutex
c := sync.NewCond(&mu)

mu.Lock()
for !ready { c.Wait() }
mu.Unlock()

mu.Lock()
ready = true
c.Broadcast()
mu.Unlock()
```

---

## 8. 자주 쓰는 패턴

### 8.1 Producer-Consumer

```c
void produce(int x) {
    lock(&m);
    while (count == N) cond_wait(&not_full, &m);
    buf[in++ % N] = x; count++;
    cond_signal(&not_empty);
    unlock(&m);
}

int consume() {
    lock(&m);
    while (count == 0) cond_wait(&not_empty, &m);
    int x = buf[out++ % N]; count--;
    cond_signal(&not_full);
    unlock(&m);
    return x;
}
```

### 8.2 Bounded Queue

```cpp
template <class T>
class BoundedQueue {
    std::queue<T> q;
    std::mutex m;
    std::condition_variable not_empty, not_full;
    size_t cap;
public:
    void push(T x) {
        std::unique_lock l(m);
        not_full.wait(l, [&]{ return q.size() < cap; });
        q.push(std::move(x));
        not_empty.notify_one();
    }
    T pop() {
        std::unique_lock l(m);
        not_empty.wait(l, [&]{ return !q.empty(); });
        T x = std::move(q.front()); q.pop();
        not_full.notify_one();
        return x;
    }
};
```

### 8.3 Shutdown 신호

```c
volatile bool shutdown = false;

void worker() {
    lock(&m);
    while (!shutdown && !task_ready)
        cond_wait(&cv, &m);
    if (shutdown) { unlock(&m); return; }
    // ...
}

void on_signal() {
    lock(&m);
    shutdown = true;
    cond_broadcast(&cv);    // 모든 워커 깨움
    unlock(&m);
}
```

---

## 9. Timed Wait

```c
struct timespec ts;
clock_gettime(CLOCK_REALTIME, &ts);
ts.tv_sec += 5;

int rc = pthread_cond_timedwait(&cv, &m, &ts);
if (rc == ETIMEDOUT) {
    // 5 초 안에 신호 못 받음
}
```

타임아웃 + 폴링 / heartbeat 용.

---

## 10. Monitor (Java synchronized)

Java 의 모든 객체 = monitor — `wait/notify/notifyAll` 메소드.

```java
synchronized (obj) {
    while (!ready) obj.wait();
    obj.notifyAll();
}
```

⚠️ `notify()` 의 의미가 미묘 — 한 명을 깨움 + 어떤 스레드인지 보장 X. 거의 항상 `notifyAll()` + while.

---

## 11. 신호 누락 (Lost Wakeup)

```c
// ❌ 깨어남이 사라짐
if (!cond) {
    // 여기서 다른 스레드가 cond 변경 + signal → 잃음
    cond_wait(&cv, &m);
}
```

cond_wait 의 atomic 성으로 막힘 — mutex 잠근 채로 wait 시작 → signal 도 mutex 잡고 발생 → 순서 보장.

→ **반드시 같은 mutex 로** 변수 수정 + signal. 변수만 atomic 으로 바꾸고 signal 분리 = lost wakeup.

---

## 12. 함정

### 12.1 `if` 사용
spurious wakeup → 가짜 진행. 항상 `while`.

### 12.2 mutex 안 잡고 signal
이론적 OK 지만 lost wakeup 위험. 잡고 signal.

### 12.3 다른 mutex 와 함께
cond 는 한 mutex 와 binding. 다른 락으로 wait = UB.

### 12.4 cond_wait 중 mutex destroy
대기자 깨움 못 받음.

### 12.5 broadcast 의 thundering herd
N 스레드가 동시에 깨어 mutex 경쟁 → 1 명만 진행, 나머지 다시 sleep. 작은 N 면 OK, 큰 N 은 design 검토.

### 12.6 cond_wait 후 mutex 만 해제
mutex 는 wait 가 자동 재획득. 의도하지 않은 unlock 추가 = double unlock.

### 12.7 무한 wait
predicate 가 영원히 false. shutdown flag + broadcast.

---

## 13. 학습 자료

- **The Little Book of Semaphores** Ch. 6
- **POSIX Threads Programming** — llnl tutorial
- `man 3 pthread_cond_wait`

---

## 14. 관련

- [[mutex]] — cond 의 짝
- [[semaphore]] — 대안
- [[classic-problems]]
- [[synchronization]] — Sync hub
