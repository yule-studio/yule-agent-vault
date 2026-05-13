---
title: "Semaphore — 세마포어"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:10:00+09:00
tags:
  - operating-system
  - sync
  - semaphore
---

# Semaphore

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Counting / Binary / POSIX |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

정수 카운터 + 두 연산:
- **P (wait, sem_wait)** — 1 감소, 0 이면 대기
- **V (post, sem_post)** — 1 증가, 대기자 깨움

Dijkstra 1965.

---

## 2. Counting vs Binary

| 종류 | 초기값 | 용도 |
| --- | --- | --- |
| **Binary** | 0 또는 1 | mutex 비슷 (1 자원) |
| **Counting** | N | N 개 자원 / N 동시 |

---

## 3. POSIX Semaphore

### 3.1 익명 (스레드 간)

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 5);              // 두번째 0 = 스레드 간; 5 = 초기값

sem_wait(&sem);                     // P
// ... critical
sem_post(&sem);                     // V

sem_trywait(&sem);                  // 즉시
sem_timedwait(&sem, &ts);
sem_getvalue(&sem, &val);

sem_destroy(&sem);
```

### 3.2 명명 (프로세스 간)

```c
sem_t *sem = sem_open("/mysem", O_CREAT, 0644, 5);
sem_wait(sem);
sem_post(sem);
sem_close(sem);
sem_unlink("/mysem");
```

`/dev/shm/sem.mysem` 에 생성.

### 3.3 System V (옛)

```c
int id = semget(key, 1, 0666 | IPC_CREAT);
struct sembuf op = {0, -1, 0};
semop(id, &op, 1);
```

복잡 / 옛. POSIX 권장.

---

## 4. 언어별

### 4.1 Java

```java
Semaphore sem = new Semaphore(5);   // counting, 5 허용

sem.acquire();
try {
    // critical
} finally {
    sem.release();
}

sem.tryAcquire(1, TimeUnit.SECONDS);
```

`fair=true` 옵션으로 FIFO.

### 4.2 Python

```python
import threading
sem = threading.Semaphore(5)
with sem:
    # critical
    pass

# 비동기
import asyncio
sem = asyncio.Semaphore(5)
async with sem: ...
```

### 4.3 C++

```cpp
#include <semaphore>      // C++20
std::counting_semaphore<10> sem(5);
sem.acquire();
sem.release();

// binary
std::binary_semaphore bsem(0);
```

### 4.4 Go

Go 는 semaphore 가 표준 X — channel 로:

```go
sem := make(chan struct{}, 5)  // 5 token

sem <- struct{}{}           // acquire
defer func() { <-sem }()     // release
```

또는 `golang.org/x/sync/semaphore`.

### 4.5 Rust

```rust
use tokio::sync::Semaphore;
let sem = Arc::new(Semaphore::new(5));
let permit = sem.acquire().await;
// drop = release
```

---

## 5. Producer-Consumer (고전 패턴)

```c
sem_t empty, full;
pthread_mutex_t m;
int buf[N], in=0, out=0;

sem_init(&empty, 0, N);     // N 빈 슬롯
sem_init(&full, 0, 0);       // 0 채워진

void producer(int item) {
    sem_wait(&empty);
    pthread_mutex_lock(&m);
    buf[in] = item; in = (in+1)%N;
    pthread_mutex_unlock(&m);
    sem_post(&full);
}

int consumer() {
    sem_wait(&full);
    pthread_mutex_lock(&m);
    int item = buf[out]; out = (out+1)%N;
    pthread_mutex_unlock(&m);
    sem_post(&empty);
    return item;
}
```

자세히 → [[classic-problems#producer-consumer]]

---

## 6. Rate Limit / 동시 제한

```python
import asyncio

sem = asyncio.Semaphore(10)        # 동시 10개

async def fetch(url):
    async with sem:
        return await http.get(url)

await asyncio.gather(*[fetch(u) for u in urls])
```

→ 동시 외부 호출 N 으로 제한.

---

## 7. Connection Pool

```
초기: counting semaphore N (= pool size)
acquire connection → sem_wait
release          → sem_post
pool 비면 sem_wait 대기
```

DB / HTTP / 외부 API 풀의 기본.

---

## 8. Barrier 비슷한 패턴

```c
sem_t done;
sem_init(&done, 0, 0);

// N 워커
work();
sem_post(&done);

// 메인
for (int i = 0; i < N; i++) sem_wait(&done);
```

→ 모든 워커 완료 대기. (전용 `pthread_barrier` 도 있음)

---

## 9. Semaphore vs Mutex

| | Mutex | Semaphore |
| --- | --- | --- |
| 의미 | 1 자원 보호 | N 자원 / signaling |
| owner | 잠근 스레드만 unlock | 누구나 post |
| 의도 | 상호 배제 | 시그널 + 자원 카운트 |
| 우선순위 상속 | 보통 지원 | 미지원 (POSIX) |

Mutex 는 owner 개념 → Priority Inheritance 가능. Semaphore 는 owner X.

→ 단순 상호 배제 = Mutex.
→ N 자원 / 시그널 = Semaphore.

---

## 10. Binary Semaphore vs Mutex

같아 보이지만:
- Mutex 는 owner — 잠근 스레드만 unlock
- Binary semaphore 는 owner X — 다른 스레드가 post 가능

⇒ 시그널 (한 스레드가 wait, 다른 스레드가 post) 패턴엔 binary semaphore. 상호 배제는 mutex.

---

## 11. Linux 의 실제

`sem_*` API 는 내부적으로 futex.
**eventfd** (`eventfd(2)`) 도 비슷한 카운터 — epoll 친화.

```c
int efd = eventfd(0, EFD_SEMAPHORE);
uint64_t one = 1;
write(efd, &one, 8);     // post
read(efd, &one, 8);       // wait (semaphore mode)
```

---

## 12. 함정

### 12.1 `sem_wait` 의 EINTR
시그널로 깨어남 → 루프 재시도 또는 SA_RESTART.

### 12.2 Buffer 사이즈 0 + sem 패턴
unbounded 자원 가정 — 실제론 메모리 폭발 가능.

### 12.3 Counting semaphore overflow
초기값 큰데 post 많이 → overflow 가능 (이론). 보통 무관.

### 12.4 sem_destroy 활성 sem
대기자 있는 sem destroy → UB.

### 12.5 Named semaphore 누수
`sem_unlink` 안 하면 OS reboot 까지 남음.

### 12.6 fork 후 익명 sem
스레드 간 (pshared=0) 은 자식 무관. 프로세스 간은 pshared=1 필요 + shared memory.

### 12.7 Mutex 대신 binary semaphore
owner 없음 → debugging / priority 문제. 명확히 mutex.

### 12.8 Spurious wakeup
실제 sem_wait 은 spurious wakeup 거의 없지만 condvar 와 자주 함께 쓰임 — 패턴은 condvar 참조.

---

## 13. 학습 자료

- **The Little Book of Semaphores** — Allen B. Downey (무료)
- **The Linux Programming Interface** Ch. 47-48
- `man 3 sem_init`, `man 7 sem_overview`

---

## 14. 관련

- [[mutex]] — 1 자원 보호
- [[condvar]] — 조건 + 락
- [[classic-problems]] — Producer-Consumer 등
- [[synchronization]] — Sync hub
