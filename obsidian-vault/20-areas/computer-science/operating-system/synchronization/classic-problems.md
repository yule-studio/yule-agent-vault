---
title: "고전 동기화 문제 — Producer-Consumer / Readers-Writers / Dining Philosophers"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:40:00+09:00
tags:
  - operating-system
  - sync
  - classic
---

# 고전 동기화 문제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 5 가지 고전 문제 |

**[[synchronization|↑ Sync hub]]**

---

## 1. Producer-Consumer (Bounded Buffer)

생산자 N + 소비자 M + 한정 버퍼.

```c
#define N 10
int buf[N], in=0, out=0;
sem_t empty, full;          // counting
pthread_mutex_t m;

sem_init(&empty, 0, N);     // 빈 슬롯 N
sem_init(&full, 0, 0);       // 채워진 슬롯 0

void produce(int item) {
    sem_wait(&empty);
    pthread_mutex_lock(&m);
    buf[in] = item;
    in = (in + 1) % N;
    pthread_mutex_unlock(&m);
    sem_post(&full);
}

int consume() {
    sem_wait(&full);
    pthread_mutex_lock(&m);
    int item = buf[out];
    out = (out + 1) % N;
    pthread_mutex_unlock(&m);
    sem_post(&empty);
    return item;
}
```

### 1.1 응용
- 작업 큐 / 메시지 큐
- 로깅 (async)
- 비동기 I/O 처리

### 1.2 condvar 버전

```c
int count = 0;
pthread_cond_t not_full, not_empty;

void produce(int item) {
    pthread_mutex_lock(&m);
    while (count == N) pthread_cond_wait(&not_full, &m);
    buf[in] = item; in = (in+1)%N; count++;
    pthread_cond_signal(&not_empty);
    pthread_mutex_unlock(&m);
}
```

---

## 2. Readers-Writers

다수 reader 동시 / writer 단독.

### 2.1 Reader-Preferring (writer starvation 위험)

```c
int readers = 0;
sem_t mutex, write_lock;

sem_init(&mutex, 0, 1);
sem_init(&write_lock, 0, 1);

void reader() {
    sem_wait(&mutex);
    readers++;
    if (readers == 1) sem_wait(&write_lock);
    sem_post(&mutex);

    /* read */

    sem_wait(&mutex);
    readers--;
    if (readers == 0) sem_post(&write_lock);
    sem_post(&mutex);
}

void writer() {
    sem_wait(&write_lock);
    /* write */
    sem_post(&write_lock);
}
```

### 2.2 Writer-Preferring

writer 대기 중이면 새 reader block.

### 2.3 Fair (FIFO)

`pthread_rwlock_t` 의 일부 구현. 자세히 → [[rwlock]]

### 2.4 응용
- 캐시 / 설정
- DB 인덱스
- 파일 시스템

---

## 3. Dining Philosophers (Dijkstra 1965)

5 명이 원탁, 각자 사이에 젓가락. 2 개 모두 잡아야 식사.

### 3.1 naive — Deadlock

```c
void philosopher(int i) {
    while (1) {
        think();
        pthread_mutex_lock(&fork[i]);            // 왼쪽
        pthread_mutex_lock(&fork[(i+1)%5]);      // 오른쪽
        eat();
        pthread_mutex_unlock(&fork[i]);
        pthread_mutex_unlock(&fork[(i+1)%5]);
    }
}
```

→ 모두가 왼쪽 잡고 오른쪽 대기 = 데드락.

### 3.2 해결 1 — 락 순서

```c
int lo = MIN(i, (i+1)%5);
int hi = MAX(i, (i+1)%5);
pthread_mutex_lock(&fork[lo]);
pthread_mutex_lock(&fork[hi]);
```

### 3.3 해결 2 — 비대칭

홀수 철학자는 왼쪽 먼저, 짝수는 오른쪽 먼저.

### 3.4 해결 3 — 동시 최대 N-1

```c
sem_t room;  // N-1 = 4
sem_wait(&room);
pthread_mutex_lock(&fork[i]);
pthread_mutex_lock(&fork[(i+1)%5]);
eat();
pthread_mutex_unlock(...);
sem_post(&room);
```

→ 항상 누군가는 2 개 잡을 수 있음.

### 3.5 해결 4 — Monitor

상태 (THINKING/HUNGRY/EATING) 관리 + condvar.

### 3.6 응용
- 자원 N 개 + 프로세스 M 개의 충돌
- 데드락 회피 패턴의 캐논

---

## 4. Sleeping Barber

이발사 1 명, 의자 N, 손님 도착.

```
손님 도착 → 대기실 자리 있으면 앉음, 없으면 떠남
이발사 → 손님 없으면 sleep, 있으면 자름
```

```c
sem_t customers;       // 대기 손님 수
sem_t barber;          // 이발사 ready
pthread_mutex_t mutex;  // 카운터 보호
int waiting = 0;

sem_init(&customers, 0, 0);
sem_init(&barber, 0, 0);

void barber_th() {
    while (1) {
        sem_wait(&customers);     // 손님 대기
        pthread_mutex_lock(&mutex);
        waiting--;
        sem_post(&barber);
        pthread_mutex_unlock(&mutex);
        cut_hair();
    }
}

void customer(int id) {
    pthread_mutex_lock(&mutex);
    if (waiting < N) {
        waiting++;
        sem_post(&customers);
        pthread_mutex_unlock(&mutex);
        sem_wait(&barber);
        get_haircut();
    } else {
        pthread_mutex_unlock(&mutex);
        leave();
    }
}
```

### 4.1 응용
- Connection pool — pool 한계 + 대기 큐
- Queue limit + reject policy

---

## 5. Cigarette Smokers (Patil 1971)

3 명의 흡연자 + 1 명의 agent.
각 흡연자는 1 가지 재료 만 무한 (담배 / 종이 / 성냥).
Agent 가 2 가지 재료 테이블에 놓음 → 부족한 1 가지 가진 자 만 흡연 후 agent 가 또 놓음.

→ 일반적 semaphore 만으로는 어려움 (helper thread 필요).

### 5.1 응용
- 미들웨어 패턴
- 학술 — semaphore 만으로 풀 수 없는 경우 보여줌

---

## 6. Bounded H2O — 분자 형성 (1980s)

H 2 + O 1 → H2O 분자.
스레드들이 자신이 H 또는 O 라 알림 → 동기화 후 형성.

```c
sem_t hsem, osem;
int waitingH = 0, waitingO = 0;
pthread_mutex_t m;

void hydrogen() {
    pthread_mutex_lock(&m);
    waitingH++;
    if (waitingH >= 2 && waitingO >= 1) {
        sem_post(&hsem); sem_post(&hsem); sem_post(&osem);
        waitingH -= 2; waitingO -= 1;
    }
    pthread_mutex_unlock(&m);
    sem_wait(&hsem);
    bond();
}

void oxygen() { /* 비슷 */ }
```

→ barrier 비슷.

---

## 7. Rendezvous

두 스레드가 서로의 어떤 지점 도달 후 진행.

```c
sem_t a_arrived, b_arrived;
sem_init(&a_arrived, 0, 0);
sem_init(&b_arrived, 0, 0);

void thread_a() {
    statement_a1();
    sem_post(&a_arrived);
    sem_wait(&b_arrived);
    statement_a2();
}
void thread_b() {
    statement_b1();
    sem_post(&b_arrived);
    sem_wait(&a_arrived);
    statement_b2();
}
```

→ `a1 < a2`, `b1 < b2`, **`a1 < b2`, `b1 < a2`** 모두 보장.

### 7.1 응용
- 두 단계 합의
- handshake

---

## 8. Reusable Barrier

```c
int count = 0;
sem_t mutex, turnstile, turnstile2;

void thread_n() {
    work();

    sem_wait(&mutex);
    count++;
    if (count == N) {
        sem_wait(&turnstile2);
        sem_post(&turnstile);
    }
    sem_post(&mutex);

    sem_wait(&turnstile);
    sem_post(&turnstile);
    // critical point — 모두 도달
    sem_wait(&mutex);
    count--;
    if (count == 0) {
        sem_wait(&turnstile);
        sem_post(&turnstile2);
    }
    sem_post(&mutex);

    sem_wait(&turnstile2);
    sem_post(&turnstile2);
}
```

POSIX `pthread_barrier_t` 가 표준 제공.

---

## 9. Lightswitch (Lock 그룹)

첫 reader 가 writer-lock 잡고, 마지막 reader 가 풀기:

```c
typedef struct { int counter; pthread_mutex_t m; } LS;

void ls_lock(LS *ls, sem_t *room_empty) {
    pthread_mutex_lock(&ls->m);
    if (++ls->counter == 1) sem_wait(room_empty);
    pthread_mutex_unlock(&ls->m);
}
void ls_unlock(LS *ls, sem_t *room_empty) {
    pthread_mutex_lock(&ls->m);
    if (--ls->counter == 0) sem_post(room_empty);
    pthread_mutex_unlock(&ls->m);
}
```

→ Readers-Writers 의 정리.

---

## 10. 실무 매핑

| 학술 문제 | 실무 |
| --- | --- |
| Producer-Consumer | 작업 큐, log buffer, Kafka producer |
| Readers-Writers | 캐시, config, RCU |
| Dining Philosophers | DB 행 락 순서 |
| Sleeping Barber | Connection pool, thread pool |
| Cigarette Smokers | 미들웨어 / 합성 라이브러리 |
| Rendezvous | 2PC, handshake |
| Barrier | Parallel batch (MPI, BSP) |

---

## 11. 학습 자료

- **The Little Book of Semaphores** — Allen B. Downey (무료, 고전 문제 집)
- **OSTEP** Ch. 31
- **Operating System Concepts** (Silberschatz) Ch. 6

---

## 12. 관련

- [[semaphore]]
- [[condvar]]
- [[deadlock]]
- [[synchronization]] — Sync hub
