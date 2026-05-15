---
title: "Distributed lock — Redlock / etcd / DB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:49:00+09:00
tags: [devops, distributed-systems, distributed-lock]
---

# Distributed lock — Redlock / etcd / DB

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 왜

```
여러 service instance / pod 가 같은 자원 동시 접근:
  - 결제 처리 (한 번만)
  - 재고 차감 (race condition)
  - cron job (1대만 실행)
  - 배치 작업
  - 외부 API rate limit (전체)

→ 단일 instance lock (synchronized) 안 됨. cross-process.
```

---

## 2. 도구 비교

| | 강점 | 약점 |
| --- | --- | --- |
| **Redis (Redlock)** | 빠름, 흔함 | Redis fail 시 lock 잃음 |
| **etcd / ZooKeeper** | strong consistency, leader election | 느림 |
| **DB row lock** | DB tx 안에서 자연 | DB 부담 |
| **Hazelcast / Ignite** | in-memory grid | infra 추가 |
| **Consul** | etcd 비슷 | |

→ **일반: Redis. critical: etcd**.

---

## 3. Redis SET NX EX (★ 단순)

```bash
SET lock:order:123 my-uuid NX EX 30

# NX: not exists 만 set
# EX 30: 30s TTL (lock holder 죽어도 자동 release)

# 성공 = lock 획득
# 실패 = 다른 process 가 잡음
```

```java
// Spring + Redis
String token = UUID.randomUUID().toString();
boolean acquired = redisTemplate.opsForValue()
    .setIfAbsent("lock:order:" + id, token, Duration.ofSeconds(30));

if (!acquired) {
    throw new LockAcquisitionException();
}

try {
    // critical section
    process();
} finally {
    // unlock — Lua script 로 atomic
    String script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
    """;
    redisTemplate.execute(
        new DefaultRedisScript<>(script, Long.class),
        List.of("lock:order:" + id),
        token
    );
}
```

→ token 검증 = 다른 process 의 lock 안 삭제.

---

## 4. Redisson (★ Java 권장)

```java
RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("lock:order:123");

// blocking
try {
    lock.lock(30, TimeUnit.SECONDS);   // 30s 안에 안 풀면 자동 release
    process();
} finally {
    lock.unlock();
}

// non-blocking
if (lock.tryLock(0, 30, TimeUnit.SECONDS)) {
    try { process(); } finally { lock.unlock(); }
}

// reentrant (재귀 가능)
// 같은 thread 가 두 번 lock OK
```

기능:
- reentrant
- pub/sub (다른 thread 가 release 즉시 알림)
- fair lock (FIFO)
- read/write lock
- watchdog (★ TTL 갱신)
- semaphore

---

## 5. Redlock 알고리즘 (★)

```
한 Redis 의 한계:
  - single point: Redis fail → lock 잃음
  - replica 의 async replication → split-brain

Redlock (multiple Redis):
  1. N개 Redis (5개 권장) 에 동시 SET
  2. (N/2 + 1) 개 success → 획득
  3. 일부 fail = 무시 (majority 만)

→ 한 Redis fail 견딤.
```

```java
// Redisson 의 redlock
RedissonClient redisson1 = ...;
RedissonClient redisson2 = ...;
RedissonClient redisson3 = ...;

RLock lock = redisson1.getMultiLock(
    redisson1.getLock("..."),
    redisson2.getLock("..."),
    redisson3.getLock("...")
);
lock.lock();
```

→ Martin Kleppmann 의 비판도 있음 (clock skew / network delay). critical 면 etcd 권장.

---

## 6. etcd distributed lock (★ strong)

```python
import etcd3

client = etcd3.client(host='etcd-1')

# lease (TTL)
lease = client.lease(60)   # 60s

# lock 시도 (lease 와 함께)
with client.lock('my-lock', ttl=60) as lock:
    # critical section
    do_work()
# 자동 release + lease 만료
```

```bash
# etcdctl
etcdctl lock /my-lock -- /bin/long-running-script
```

→ Raft 기반 → strong consistency. split-brain 절대 없음.

---

## 7. DB row lock (★ tx 안)

```sql
-- 가장 간단
BEGIN;
SELECT * FROM orders WHERE id = 123 FOR UPDATE;
-- 다른 tx 는 이 row 를 read 못 함

-- 작업
UPDATE orders SET status = 'PROCESSING' WHERE id = 123;

COMMIT;
```

```sql
-- queue 패턴 (SKIP LOCKED)
SELECT * FROM jobs 
WHERE status = 'PENDING' 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
-- 다른 worker 가 잡은 row 는 skip
-- → 자연스러운 work distribution
```

→ 단순. DB tx 와 동일 boundary. 부담 ↑.

---

## 8. lock 의 정확성

```
"correct" distributed lock 어렵다 (Kleppmann):

1. clock drift
   서버 간 시간 차 → TTL 의미 다름
   
2. process pause
   GC pause / hypervisor → lock 만료 인식 못 함
   
3. network partition
   master ↔ replica 끊김 → 두 lock 동시 가능

→ critical 한 곳:
   - fencing token 사용 (★)
   - idempotency
   - 외부 storage 의 conditional write
```

---

## 9. fencing token (★)

```
lock 받을 때 token 발급 (단조 증가):
  process A: token=5
  process B (A 의 GC pause 후): token=6
  
→ external storage 에 write 시 token 첨부:
  storage 가 token 검증
  옛 token (5) 의 write 거부

→ "분산 lock" 만으로 부족. fencing 같이.
```

```python
# Zookeeper / etcd 의 sequential znode = token
# Redis 의 INCR = token

token = redis.incr("lock-token")

# 외부 system 에 write 시
storage.write(data, fencing_token=token)
```

---

## 10. lock 의 패턴

### A. critical section

```
한 사람 만 critical section 안.
일반적인 mutual exclusion.
```

### B. leader election

```
N node 중 1명이 leader.
다른 N-1 은 follower (idle).
leader fail → 다른 node 가 leader.

도구: etcd / ZooKeeper 의 ephemeral node.
```

### C. quota / rate limit

```
"동시 5명 만 처리 OK" (semaphore).

Redisson:
  RSemaphore semaphore = redisson.getSemaphore("api");
  semaphore.trySetPermits(5);
  semaphore.acquire();
  ...
  semaphore.release();
```

### D. cron singleton

```
N pod 중 1개만 cron 실행:
  ShedLock (Spring), Quartz cluster mode
```

```java
@Scheduled(cron = "0 0 * * * *")
@SchedulerLock(name = "daily-report", lockAtLeastFor = "PT5M", lockAtMostFor = "PT15M")
public void dailyReport() {
    // ShedLock 이 DB / Redis 에 lock
    // → N pod 중 1개만 실행
}
```

---

## 11. anti-patterns

```
❌ TTL 없는 lock — process 죽으면 영원 lock
❌ unlock 검증 안 함 — 다른 process 의 lock 삭제
❌ critical section 너무 김 — TTL 안에 못 끝남
❌ retry 없는 acquire — 한 번 fail = 영원 fail
❌ lock 잡고 외부 API 호출 — 외부가 lock 잡힘
❌ recursive lock 무지 — deadlock
❌ "분산 lock = 안전" 가정 — fencing token 필요
```

---

## 12. 함정

1. **Redis single-instance lock** — fail 시 잃음. Redlock 또는 etcd.
2. **TTL 너무 짧음** — critical section 안 끝나면 다른 process 가 잡음.
3. **TTL 너무 김** — process 죽으면 오래 lock.
4. **unlock 의 race** — token 검증 + atomic Lua.
5. **watchdog 없음** — Redisson 의 자동 갱신 활용.
6. **fencing 없음** — 진짜 critical 면 token.
7. **DB lock vs Redis lock 혼동** — DB tx 안이면 row lock 권장.

---

## 13. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[consensus-raft]]
- [[idempotency]]
- [[../../60-recipes/spring/distributed-lock/distributed-lock|↗ Redisson recipe]]
