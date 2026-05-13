---
title: "Threads (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:30:00+09:00
tags:
  - operating-system
  - thread
  - hub
---

# Threads (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 4 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

스레드 = **같은 프로세스 안의 경량 실행 단위**. 주소 공간 공유, stack/register 만 자신만의 것.

---

## 2. 프로세스 vs 스레드

| 항목 | Process | Thread |
| --- | --- | --- |
| 주소 공간 | 독립 | 공유 |
| Heap / Global | 독립 | 공유 |
| Stack | 독립 | 독립 |
| FD table | 독립 | 공유 |
| 생성 비용 | 큼 | 작음 |
| 컨텍스트 스위치 | 무거움 (TLB flush) | 가벼움 |
| 통신 | IPC | 메모리 직접 |
| 크래시 | 격리 | 한 스레드 죽으면 전체 |

---

## 3. POSIX Threads (pthread)

```c
#include <pthread.h>

void *worker(void *arg) {
    int id = (intptr_t)arg;
    printf("thread %d\n", id);
    return NULL;
}

int main() {
    pthread_t t[4];
    for (int i = 0; i < 4; i++)
        pthread_create(&t[i], NULL, worker, (void*)(intptr_t)i);
    for (int i = 0; i < 4; i++)
        pthread_join(t[i], NULL);
}
```

### 3.1 자주 쓰는 함수

```c
pthread_create(&t, attr, fn, arg);
pthread_join(t, &retval);          // 대기
pthread_detach(t);                  // 자동 회수
pthread_exit(retval);
pthread_self();
pthread_equal(t1, t2);
pthread_cancel(t);                  // 신중히 — cleanup handler 필요
```

---

## 4. Linux 의 스레드 = task

Linux 에서 스레드 = mm 공유하는 task. `clone(CLONE_VM | CLONE_FILES | CLONE_SIGHAND | ...)`.

```c
clone(child_fn, stack, CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD, arg);
```

→ task_struct 가 같은 mm_struct 를 가리킴. 자세히 → [[../process/pcb]]

---

## 5. 스레드 모델

| 모델 | 설명 |
| --- | --- |
| **1:1 (Kernel)** | 사용자 스레드 1 = 커널 스레드 1. Linux/Windows |
| **N:1 (User)** | 사용자 N : 커널 1. blocking syscall = 전체 stop |
| **N:M (Hybrid)** | 사용자 N : 커널 M. Go 의 goroutine, Erlang |

자세히 → [[thread-models]]

---

## 6. 사용자 / 커널 스레드

자세히 → [[kernel-vs-user-thread]]

---

## 7. Thread Local Storage (TLS)

```c
__thread int counter;          // C: 스레드별 변수

pthread_key_t key;
pthread_key_create(&key, destructor);
pthread_setspecific(key, ptr);
pthread_getspecific(key);
```

각 스레드 = 자기만의 변수. 락 없이 안전.

자세히 → [[tls]]

---

## 8. Thread Pool

자세히 → [[thread-pool]]

스레드 생성 / 종료 비용 절약 + 동시성 제한.

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=16) as pool:
    futures = [pool.submit(work, x) for x in items]
    for f in futures:
        print(f.result())
```

---

## 9. 동기화

스레드는 메모리 공유 → race condition. 자세히 → [[../synchronization/synchronization]]

```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&m);
shared++;
pthread_mutex_unlock(&m);
```

---

## 10. fork 와 스레드 — 위험

자세히 → [[../process/fork-exec#9-fork-와-멀티스레드의-위험]]

→ fork 후 즉시 exec 또는 멀티스레드 + fork 회피.

---

## 11. Async / Coroutine vs Thread

| 모델 | 단위 |
| --- | --- |
| Thread | OS 가 스케줄. ms 단위 stack (1-8 MB) |
| Coroutine / async | 응용이 스케줄. KB 단위 stack |
| Go goroutine | KB stack + N:M |
| Java virtual thread (21+) | 경량 |
| Rust async / Tokio | future-based |

I/O bound 작업 → async 가 효율 ↑.
CPU bound → 진짜 스레드 (코어 수 만큼).

---

## 12. 스레드 갯수 — 적정

| 워크로드 | 권장 |
| --- | --- |
| CPU bound | 코어 수 |
| I/O bound | 코어 수 × 2 ~ 10 |
| 혼합 | 측정 |
| 단순 fan-out | thread pool 16-64 |

너무 많은 스레드:
- stack 메모리 폭발 (1024 × 8MB = 8GB)
- 컨텍스트 스위치 폭증
- 락 경합

---

## 13. Linux 에서 보기

```bash
ps -eLf | grep myapp                 # 스레드 보기
ps -L -p $PID                         # 특정 프로세스의 스레드
top -H                                 # 스레드 단위 view
htop                                   # 'H' 키로 스레드 표시

cat /proc/$PID/task/                  # 스레드 목록
cat /proc/$PID/status | grep Threads

# 스레드 CPU
pidstat -t -p $PID 1
```

---

## 14. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[thread-models]] | 1:1 / N:1 / N:M |
| [[kernel-vs-user-thread]] | 차이 + 구현 |
| [[thread-pool]] | pool 패턴 + 라이브러리 |
| [[tls]] | Thread Local Storage |

---

## 15. 함정

### 함정 1 — fork + threads
잠긴 락 영원히. 자세히 → [[../process/fork-exec]]

### 함정 2 — 스레드 누수
pthread_detach 안 함 + join 안 함 → 좀비처럼 누적.

### 함정 3 — 너무 많은 스레드
stack × N = GB. 풀로 제한.

### 함정 4 — Cancellation
`pthread_cancel` 은 cleanup 안 보장. async-cancel-safe 함수만.

### 함정 5 — TLS leak
`__thread` 변수에 큰 buffer — 모든 스레드 곱하기.

### 함정 6 — Volatile 만으로 sync 가정
C/C++ 의 volatile 은 atomic X. atomic / fence 사용.

### 함정 7 — GIL (Python)
CPython 은 GIL 로 한 번에 하나만 실행. multiprocessing / 진짜 native ext.

---

## 16. 학습 자료

- **The Linux Programming Interface** Ch. 29-33 (Threads)
- **C++ Concurrency in Action** — Williams
- **Java Concurrency in Practice** — Goetz
- **pthread tutorial** — computing.llnl.gov

---

## 17. 관련

- [[../process/process]] — Process 비교
- [[../synchronization/synchronization]] — race / lock
- [[../scheduling/scheduling]] — 스레드 스케줄링
- [[../operating-system|↑ OS hub]]
