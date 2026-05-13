---
title: "Thread Pool — 스레드 풀 패턴"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:45:00+09:00
tags:
  - operating-system
  - thread
  - pool
---

# Thread Pool

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 풀 패턴 + 라이브러리 |

**[[threads|↑ Threads hub]]**

---

## 1. 왜 풀

```
매번 pthread_create + pthread_join
→ 생성/소멸 비용 + 메모리 폭증
→ 풀로 재사용
```

장점:
- 생성 비용 amortize
- 동시성 상한 (메모리 / 외부 자원 보호)
- 큐로 backpressure

---

## 2. 기본 구조

```
[Task Queue] ← submit()
     ↓
[Worker 1] [Worker 2] ... [Worker N]
     ↓             ↓
   process       process
```

- N 명의 worker = 같은 큐 polling
- 큐 비면 condvar 대기
- shutdown 시 모두 join

---

## 3. C 로 작은 예 (개념)

```c
typedef struct { void (*fn)(void*); void *arg; } Task;

pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cv  = PTHREAD_COND_INITIALIZER;
Task queue[QSIZE]; int qlen = 0;
int shutdown_flag = 0;

void *worker(void *_) {
    while (1) {
        pthread_mutex_lock(&mtx);
        while (qlen == 0 && !shutdown_flag) pthread_cond_wait(&cv, &mtx);
        if (shutdown_flag && qlen == 0) { pthread_mutex_unlock(&mtx); break; }
        Task t = queue[--qlen];
        pthread_mutex_unlock(&mtx);
        t.fn(t.arg);
    }
    return NULL;
}

void submit(void (*fn)(void*), void *arg) {
    pthread_mutex_lock(&mtx);
    queue[qlen++] = (Task){fn, arg};
    pthread_cond_signal(&cv);
    pthread_mutex_unlock(&mtx);
}
```

---

## 4. 라이브러리 표준

### 4.1 Java

```java
ExecutorService pool = Executors.newFixedThreadPool(16);
Future<Integer> f = pool.submit(() -> work());
int result = f.get();
pool.shutdown();
pool.awaitTermination(60, TimeUnit.SECONDS);
```

- `newFixedThreadPool(n)`
- `newCachedThreadPool()` — 동적 (위험 — 폭증)
- `newScheduledThreadPool(n)` — cron-like
- `newWorkStealingPool()` — ForkJoinPool
- `newVirtualThreadPerTaskExecutor()` — Loom

### 4.2 Python

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=32) as pool:
    futures = [pool.submit(work, x) for x in items]
    for f in as_completed(futures):
        print(f.result())
```

### 4.3 Go

Go 는 풀 없음 — `go func() {}` 가 가벼움. 필요시 `chan` + worker:

```go
jobs := make(chan int, 100)
for w := 0; w < 8; w++ {
    go func() { for j := range jobs { work(j) } }()
}
for _, j := range items { jobs <- j }
close(jobs)
```

### 4.4 C++

```cpp
#include <future>
auto f = std::async(std::launch::async, work);
int r = f.get();

// 풀은 표준 X — boost::thread_pool / Folly / Asio
```

### 4.5 Rust

```rust
use rayon::prelude::*;
items.par_iter().for_each(|x| work(x));

// 또는 tokio::task::spawn_blocking
```

---

## 5. 풀 크기 — 어떻게 정하나

### 5.1 CPU bound
```
풀 크기 = 코어 수
```

### 5.2 I/O bound — Brian Goetz 공식

```
N = N_cpu × U_cpu × (1 + W/C)

N_cpu = 코어 수
U_cpu = 목표 사용률 (0~1)
W/C   = wait time / compute time
```

예: 8 코어, target 80%, I/O = 90% of time → W/C = 9.
N = 8 × 0.8 × 10 = 64.

### 5.3 측정이 진실
profiling 후 조정. 너무 많으면 컨텍스트 스위치 폭증.

---

## 6. 큐 정책

```
무한 큐 — backpressure X → OOM 위험
유한 큐 + reject policy:
  - Abort (예외)
  - Caller runs (호출자가 직접 실행 — backpressure)
  - Discard (조용히 drop)
  - Discard oldest (오래된 것 drop)
```

Java `ThreadPoolExecutor` 의 `RejectedExecutionHandler`.

---

## 7. Work-Stealing

```
각 worker 마다 deque
빈 worker 가 다른 worker 의 deque 끝에서 훔침
```

- Java `ForkJoinPool`
- Tokio
- .NET ThreadPool
- 부하 균형 ↑

---

## 8. 자주 보는 풀 패턴

### 8.1 Fan-out / Fan-in

```
1 job → split → N workers → merge
```

### 8.2 Pipeline

```
[Stage 1 pool] → [Stage 2 pool] → [Stage 3 pool]
```

각 stage 의 풀 크기 다를 수 있음.

### 8.3 Worker per CPU

```
spawn 1 worker per core, pin to that core
SO_REUSEPORT 와 결합 (network)
```

### 8.4 Backpressure

```
큐 가득 → 호출자 block / drop / 429 응답
```

---

## 9. 셧다운

```java
pool.shutdown();             // 새 submit 거부, 기존 작업 완료
pool.shutdownNow();          // interrupt
pool.awaitTermination(30, SECONDS);
```

graceful shutdown 패턴:
1. signal (SIGTERM) 수신
2. 새 작업 거부
3. queue 비울 때까지 대기 (timeout)
4. 안 끝나면 강제 종료

---

## 10. ThreadLocal + Pool 함정

```java
ThreadLocal<DB> conn = new ThreadLocal<>();
```

풀의 스레드가 재사용됨 → ThreadLocal 의 값이 다음 task 에 누수.
→ try/finally 로 remove() 필수.

---

## 11. Connection Pool 과의 관계

DB connection pool / HTTP pool 도 같은 패턴 — 자원 풀.

```
스레드 풀:   thread × N
연결 풀:     connection × M
파일 디스크립터 풀 등도 동일 사고
```

---

## 12. 함정

### 12.1 무한 큐
OOM. 유한 + reject policy.

### 12.2 newCachedThreadPool (Java)
무한 스레드 — burst 시 폭증.

### 12.3 풀 크기 = thread = 큐 무한
OOM. 큐 / 거절 둘 다 설정.

### 12.4 ThreadLocal leak
task 끝에 clear. Tomcat 의 흔한 메모리 누수.

### 12.5 Shutdown 누락
JVM 종료 안 됨 (non-daemon thread).

### 12.6 Blocking 한 task 가 풀 점유
큐 backup → 다른 작업 대기. Timeout 강제.

### 12.7 Recursive task 데드락
같은 풀에 submit + get → 자기 자신 기다림. ForkJoin 또는 별도 풀.

### 12.8 풀 안에서 외부 호출 hang
모든 worker 잠김 → 새 task 불가. Bulkhead 패턴.

---

## 13. 학습 자료

- **Java Concurrency in Practice** — Brian Goetz
- **C++ Concurrency in Action** Ch. 9
- **The Art of Multiprocessor Programming**

---

## 14. 관련

- [[../synchronization/synchronization]] — 큐 + condvar
- [[../io/io]] — I/O bound + thread pool
- [[threads]] — Threads hub
