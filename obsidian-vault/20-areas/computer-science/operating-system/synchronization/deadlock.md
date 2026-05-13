---
title: "Deadlock — 데드락"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:35:00+09:00
tags:
  - operating-system
  - sync
  - deadlock
---

# Deadlock

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 4 조건 / 회피 / 검출 |

**[[synchronization|↑ Sync hub]]**

---

## 1. 한 줄

여러 스레드가 **순환 대기** 로 무한히 진행 못 함.

```
T1: lock A → wait B
T2: lock B → wait A
→ 영원히 멈춤
```

---

## 2. Coffman 의 4 조건 (1971)

데드락 = 다음 4 가지가 **동시에** 만족:

1. **Mutual Exclusion** — 자원은 한 번에 하나만
2. **Hold and Wait** — 잡고 다른 것 대기
3. **No Preemption** — 강제 회수 X
4. **Circular Wait** — 순환 대기 그래프

→ 하나라도 깨면 데드락 없음.

---

## 3. 예

```c
pthread_mutex_t A, B;

void *t1(void *_) {
    pthread_mutex_lock(&A);
    sleep(1);
    pthread_mutex_lock(&B);   // 여기서 멈춤
    // ...
}
void *t2(void *_) {
    pthread_mutex_lock(&B);
    sleep(1);
    pthread_mutex_lock(&A);   // 여기서 멈춤
    // ...
}
```

---

## 4. 회피 — 락 순서 고정

```c
// 모든 스레드가 동일 순서로 잡는다
void take_two(Account *a, Account *b) {
    Account *first  = (a < b) ? a : b;
    Account *second = (a < b) ? b : a;
    pthread_mutex_lock(&first->m);
    pthread_mutex_lock(&second->m);
    // ...
}
```

→ 가장 흔한 / 실용적 해결.

---

## 5. try_lock + 백오프

```c
while (1) {
    pthread_mutex_lock(&A);
    if (pthread_mutex_trylock(&B) == 0) break;
    pthread_mutex_unlock(&A);
    usleep(random() % 100);     // 백오프 + jitter
}
```

→ 락 순서 정할 수 없을 때.

---

## 6. Timeout

```c
struct timespec ts;
clock_gettime(CLOCK_REALTIME, &ts);
ts.tv_sec += 5;
if (pthread_mutex_timedlock(&B, &ts) != 0) {
    pthread_mutex_unlock(&A);
    // 데드락 감지 — 재시도 / 에러
}
```

5 초 후 포기 — SLO 보장 + 데드락 회복.

---

## 7. Banker's Algorithm (회피)

자원 할당 전 안전 상태 (safe state) 검증.

```
시스템은 모든 가능한 미래의 실행 순서에서 모든 프로세스가 끝날 수 있는가?
→ 예: 할당
→ 아니오: 거절
```

실용성 ↓ (자원 수 / max 알아야) — 학술 / 임베디드 RTOS 정도.

---

## 8. 데드락 검출 + 복구

```
주기적 자원 할당 그래프 검사 → cycle 발견
→ 한 프로세스 강제 종료 / 자원 회수
```

OS / DB 가 사용:
- PostgreSQL — `deadlock_timeout` (1s 기본) 후 검사 → 한 쪽 abort
- MySQL — InnoDB 자동 감지 + rollback
- Linux 커널 — `lockdep` 으로 정적 검출

---

## 9. 무시 (Ostrich Algorithm)

대부분의 OS (Linux, Windows) 응용 데드락 = 무시.
응용 책임. 죽으면 죽는 거.

→ Tanenbaum 의 농담 + 현실.

---

## 10. 정적 검출 — Lockdep (Linux)

Linux 커널의 lock 의존성 그래프 검사. 부팅 / 테스트 시 잠재 데드락 발견.

```
make CONFIG_LOCKDEP=y
```

응용에선 ThreadSanitizer (`-fsanitize=thread`) 가 비슷.

---

## 11. Distributed Deadlock

여러 노드 사이의 데드락:
- DB 의 cross-shard 트랜잭션
- 분산 락 (Redis / ZooKeeper)

해결:
- Timeout
- Wound-Wait / Wait-Die (rank 기반)
- 중앙 deadlock detector (DB)

---

## 12. Livelock

```
T1: lock A → 보고 T2 양보 → unlock → 다시 시도
T2: lock B → 보고 T1 양보 → unlock → 다시 시도
→ 영원히 양보
```

데드락 X (진행 시도) 이지만 일 진행 0.
→ 랜덤 백오프 + jitter.

---

## 13. Priority Inversion

[[mutex#7-priority-inversion]] 참조.

데드락 아니지만 우선순위 역전 — Mars Pathfinder 의 유명 사례.

---

## 14. Starvation

특정 스레드가 무한 대기. 데드락 X (전체는 진행).
해결: aging, fair scheduling, FIFO lock.

---

## 15. 진단

### 15.1 pstack / gdb thread apply all bt

```bash
gdb -p $PID
(gdb) thread apply all bt
```

각 스레드의 스택. lock 대기 함수 (`__lll_lock_wait`) 가 보임.

### 15.2 ThreadSanitizer

```bash
g++ -fsanitize=thread -g app.cc
./a.out
# Race / deadlock 보고
```

### 15.3 Java jstack / jconsole

```bash
jstack $PID | grep -A5 "BLOCKED"
```

JVM 은 데드락 자동 감지 + 리포트.

### 15.4 PostgreSQL

```sql
SELECT * FROM pg_locks WHERE NOT granted;
SELECT pg_blocking_pids(pid), * FROM pg_stat_activity;
```

자세히 → [[../../database/postgresql/transactions-mvcc#6-lock]]

---

## 16. 자주 보는 패턴

### 16.1 Lock + 외부 호출

```c
lock(&m);
callback(...);     // 외부 코드가 다른 lock 시도 → 데드락
unlock(&m);
```

→ lock 밖에서 callback.

### 16.2 두 객체의 메소드

```cpp
class Account {
    void transfer(Account& other, int amount) {
        std::lock_guard l1(this->m);
        std::lock_guard l2(other.m);
        ...
    }
};

a.transfer(b, 100);
b.transfer(a, 100);   // → 데드락
```

→ `std::lock(this->m, other.m)` 가 atomic 으로 둘 다 잡음.

```cpp
std::lock(this->m, other.m);
std::lock_guard l1(this->m, std::adopt_lock);
std::lock_guard l2(other.m, std::adopt_lock);
```

---

## 17. DB 데드락

```sql
T1: UPDATE acc WHERE id=1;
T2: UPDATE acc WHERE id=2;
T1: UPDATE acc WHERE id=2;  ← T2 락 대기
T2: UPDATE acc WHERE id=1;  ← T1 락 대기 → DEADLOCK
```

DB 가 자동 감지 + abort. 응용은 재시도 로직.

```python
for attempt in range(5):
    try:
        with conn.transaction():
            ...
        break
    except DeadlockError:
        time.sleep(0.01 * 2 ** attempt)
```

---

## 18. 함정

### 18.1 락 순서 가정
변경 시 깨짐. 코드 컨벤션 + 리뷰.

### 18.2 nested lock
한 함수가 잡고, 다른 함수 호출 시 다른 락 — 트리 구조 / DAG.

### 18.3 callback 안에 lock
외부 코드 호출 시 lock 해제.

### 18.4 lock 안에서 sleep / I/O
다른 스레드 모두 대기 + tail latency 폭증.

### 18.5 RAII 없는 unlock 누락
exception path → 영원히 락.

### 18.6 condvar + wrong mutex
[[condvar#12-함정]] 참조.

### 18.7 분산 락의 단일 시점
Redis 락 만료 + 작업 진행 중 → 다른 노드가 락 획득 = 동시 실행. fencing token.

---

## 19. 학습 자료

- **OSTEP** Ch. 32
- **Java Concurrency in Practice** Ch. 10
- **C++ Concurrency in Action** Ch. 3
- Linux `Documentation/locking/lockdep-design.rst`

---

## 20. 관련

- [[mutex]] — 락 순서
- [[../../database/postgresql/transactions-mvcc]] — DB 락
- [[../../database/redis/distributed-lock]] — 분산 락
- [[synchronization]] — Sync hub
