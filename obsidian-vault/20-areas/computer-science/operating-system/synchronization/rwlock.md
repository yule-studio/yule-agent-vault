---
title: "Read-Write Lock — RW 락"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:20:00+09:00
tags:
  - operating-system
  - sync
  - rwlock
---

# Read-Write Lock

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | shared / exclusive / starvation |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

**reader 다수 동시, writer 단독**.
읽기가 압도적으로 많은 자원에 효율.

---

## 2. 규칙

| 보유 중 | 새 reader | 새 writer |
| --- | --- | --- |
| 아무도 X | ✅ | ✅ |
| Reader N | ✅ | 대기 |
| Writer 1 | 대기 | 대기 |

---

## 3. POSIX

```c
pthread_rwlock_t rw = PTHREAD_RWLOCK_INITIALIZER;

pthread_rwlock_rdlock(&rw);     // 읽기 잠금
// read
pthread_rwlock_unlock(&rw);

pthread_rwlock_wrlock(&rw);     // 쓰기 잠금
// write
pthread_rwlock_unlock(&rw);

pthread_rwlock_tryrdlock(&rw);
pthread_rwlock_trywrlock(&rw);
pthread_rwlock_timedrdlock(&rw, &ts);

pthread_rwlock_destroy(&rw);
```

---

## 4. 언어별

### 4.1 C++

```cpp
#include <shared_mutex>      // C++17
std::shared_mutex rw;

// read
{
    std::shared_lock lock(rw);
    // ...
}

// write
{
    std::unique_lock lock(rw);
    // ...
}
```

### 4.2 Java

```java
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();
try { ... } finally { rw.readLock().unlock(); }

rw.writeLock().lock();
try { ... } finally { rw.writeLock().unlock(); }
```

8+: `StampedLock` (낙관적 read 가능).

### 4.3 Python

```python
# 표준 X — `readerwriterlock` 라이브러리 또는 직접
```

### 4.4 Rust

```rust
use std::sync::RwLock;

let rw = RwLock::new(0);
let r = rw.read().unwrap();
// drop = release

let mut w = rw.write().unwrap();
*w += 1;
```

### 4.5 Go

```go
var rw sync.RWMutex
rw.RLock(); /* read */; rw.RUnlock()
rw.Lock();  /* write */; rw.Unlock()
```

---

## 5. Starvation 정책

### 5.1 Reader-Preferring
reader 가 연속 진입 → writer 무한 대기 가능.

### 5.2 Writer-Preferring
writer 대기 중이면 새 reader 차단.

### 5.3 Fair (FIFO)
도착 순서.

POSIX 는 OS / glibc 구현 의존. Java `ReentrantReadWriteLock` 은 `fair=true` 옵션.

→ writer 가 자주 등장하면 reader-preferring 위험.

---

## 6. 비용 vs 효과

```
RWLock 의 acquire 는 mutex 보다 무거움 (atomic CAS + counter)
→ critical section 이 충분히 길고 read 비율 높을 때만 유리
```

실험: **mutex 가 RWLock 보다 빠른 경우 흔함** — short critical section.

---

## 7. Upgrade / Downgrade

```cpp
// C++17 — upgrade 직접 지원 X
std::shared_lock<std::shared_mutex> r(rw);
// upgrade 필요하면 unlock + wrlock (race window!)
r.unlock();
{
    std::unique_lock<std::shared_mutex> w(rw);
    // write
}
```

```java
// Java — downgrade O, upgrade X (deadlock 위험)
rw.writeLock().lock();
... write ...
rw.readLock().lock();    // downgrade
rw.writeLock().unlock();
... read ...
rw.readLock().unlock();
```

⚠️ Upgrade (read → write) 는 두 reader 가 동시에 시도하면 deadlock — 일반적으로 unsupported.

---

## 8. StampedLock (Java 8+)

```java
StampedLock sl = new StampedLock();

// 낙관적 read
long stamp = sl.tryOptimisticRead();
int v = data;
if (!sl.validate(stamp)) {
    stamp = sl.readLock();
    try { v = data; } finally { sl.unlockRead(stamp); }
}
```

writer 가 없으면 lock 없이 read — 매우 빠름.

---

## 9. 적용 시나리오

| 시나리오 | RWLock? |
| --- | --- |
| Config / Cache (가끔 update, 자주 read) | ✅ |
| 큰 자료구조 (HashMap) | ⚠️ ConcurrentHashMap 이 보통 더 빠름 |
| 빈번한 update + 짧은 read | ❌ mutex |
| 매우 짧은 read | ❌ atomic / RCU |

---

## 10. RCU (Read-Copy-Update)

Linux 커널의 대표 lock-free read.

```
Reader 는 락 없이 읽음
Writer 가 새 버전 만들고 atomic 으로 포인터 swap
옛 버전은 모든 reader 끝난 후 자유롭게 free
```

→ Reader latency 일정 + 거의 0.
Userspace 는 liburcu, RCU-like 패턴.

---

## 11. ConcurrentHashMap 등 lock-free

Java `ConcurrentHashMap` — 세분화된 락 + CAS. 대체로 RWLock 보다 빠름.
Rust `parking_lot::RwLock` 도 표준보다 빠름.

⇒ "RWLock 이 답" 보다 **lock-free 자료구조 우선** 검토.

---

## 12. 함정

### 12.1 짧은 critical section 에 RWLock
mutex 가 더 빠를 수 있음. 측정.

### 12.2 Writer starvation
reader 가 끊임없이 들어옴. fair 모드.

### 12.3 Reader starvation
writer 가 자주 → reader-preferring 으로 해결, 또는 batching.

### 12.4 Upgrade
read → write deadlock. 처음부터 write.

### 12.5 read 중 modification
read lock 잡고 read 만 — write 하면 race.

### 12.6 RAII 누락
unlock 누락 → permanent block.

### 12.7 Lock-free 대안 미고려
ConcurrentHashMap / atomic / RCU.

---

## 13. 학습 자료

- **The Art of Multiprocessor Programming** Ch. 8
- **Java Concurrency in Practice** Ch. 13
- `man 3 pthread_rwlock`
- Linux RCU documentation

---

## 14. 관련

- [[mutex]]
- [[atomic]] — lock-free 대안
- [[synchronization]] — Sync hub
