---
title: "Redis 분산 락 — SETNX / Redlock / 만료"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:55:00+09:00
tags:
  - database
  - redis
  - lock
  - distributed
---

# Redis 분산 락

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SET NX / 만료 / Redlock |

**[[redis|↑ Redis hub]]**

---

## 1. 분산 락이 필요한 이유

```
다중 워커 / 인스턴스가 같은 리소스 (DB row / 외부 API) 를 동시에 처리 X
→ 락
```

대안:
- DB 락 (`SELECT FOR UPDATE`)
- Zookeeper / etcd 의 합의 기반
- **Redis 의 빠른 in-memory 락** (이 노트)

---

## 2. 기본 패턴 — SET NX EX

```python
# 락 획득
token = uuid4().hex
acquired = r.set(f"lock:order:{id}", token, nx=True, ex=30)

if acquired:
    try:
        do_work()
    finally:
        # 안전한 해제 — 내가 잡은 것만 해제
        r.eval("""
            if redis.call('GET', KEYS[1]) == ARGV[1] then
              return redis.call('DEL', KEYS[1])
            else
              return 0
            end
        """, 1, f"lock:order:{id}", token)
else:
    # 락 못 잡음 — wait / retry / skip
    pass
```

### 2.1 왜 `SET NX EX`
- `NX` — 없을 때만 → 락 의미
- `EX` — TTL → 워커가 죽어도 자동 해제 (deadlock 방지)
- `token` — 다른 워커가 실수로 내 락 해제 못 함

### 2.2 왜 `EVAL DEL`
GET + DEL 두 명령 사이에 다른 클라이언트가 락 만료 후 새로 잡을 수 있음 (race). Lua 로 원자성.

---

## 3. TTL 의 함정

```
TTL = 30 초로 설정
실제 작업이 31 초 걸림
→ 30 초에 락 만료
→ 다른 워커가 락 획득
→ 동시 실행 (락 의미 X)
```

### 3.1 대응

1. **TTL 충분히 길게** — 작업 99 분위보다 크게.
2. **Lock extension (watchdog)** — 작업 진행 중 주기적 PEXPIRE.
3. **Fencing token** — 모노토닉 토큰을 외부에 전달 → 외부가 옛 토큰 거부.

```python
# watchdog
def renew():
    while still_working:
        time.sleep(10)
        r.pexpire(key, 30000, xx=True)
```

### 3.2 Fencing Token

```
1. Redis 락 + INCR fence_counter → token
2. 작업 호출 시 token 전달
3. 외부 (예: storage) 가 마지막으로 본 token 보다 작으면 거부
→ 옛 워커의 변경을 막음
```

---

## 4. Redlock (멀티 Redis)

단일 Redis 의 장애 → 락 손실. Redlock = N 개 (홀수, 보통 5) 의 독립 Redis 인스턴스에 락 획득. 과반수 (3/5) 성공 시 락 보유.

```
1. 모든 N 개에 SET NX EX 시도 (timeout 짧게)
2. 과반수 성공 + 사용한 시간 < TTL → 락 보유
3. 실패 시 모두 release
```

### 4.1 토론
- **Martin Kleppmann** — Redlock 안전 X (clock drift, GC pause, network partition).
- **antirez** — fencing token 보조 시 안전.
- **결론** — 진짜 안전한 락은 합의 기반 (etcd / Zookeeper). Redlock 은 빠르지만 완벽 X.

### 4.2 라이브러리

- **Redisson** (Java) — 표준 구현
- **redlock-py** (Python)
- **redsync** (Go)

---

## 5. 일반 락이 적합한 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 백오피스 작업 1회만 | ✅ |
| 외부 API 호출 dedup | ✅ |
| Cron job singleton | ✅ |
| 큐 작업 중복 처리 방지 | ✅ |
| 금융 트랜잭션 | ⚠️ — DB row lock |
| Multi-DC 강한 일관성 | ❌ — 합의 기반 |

---

## 6. DB 락 vs Redis 락

| 항목 | DB SELECT FOR UPDATE | Redis 락 |
| --- | --- | --- |
| 일관성 | 강 | 강 (단일 노드) |
| 성능 | 보통 | 매우 빠름 |
| 정전 시 | DB 의 보장 | TTL 만료까지 |
| 다중 리소스 | 트랜잭션 | 여러 키 |
| 권장 | 같은 DB 내 자원 | 외부 자원 / cross-service |

---

## 7. SETNX (옛) vs SET NX

`SETNX key val` 은 옛 명령. TTL 없음. **`SET key val NX EX ttl`** 사용.

---

## 8. SET 옵션 정리

```
SET key value [EX seconds | PX ms | EXAT ts | PXAT ms]
              [NX | XX] [KEEPTTL] [GET]
```

| 옵션 | 의미 |
| --- | --- |
| `NX` | 없을 때만 (락) |
| `XX` | 있을 때만 |
| `EX/PX` | TTL (초/ms) |
| `EXAT/PXAT` | 절대 시간 |
| `KEEPTTL` | 기존 TTL 유지 |
| `GET` | 옛 값 반환 (6.2+) |

---

## 9. 권한 / 카운터 락

### 9.1 단순 mutex

```python
def with_lock(key, ttl, fn):
    token = uuid4().hex
    if r.set(f"lock:{key}", token, nx=True, ex=ttl):
        try:
            return fn()
        finally:
            r.eval(LUA_RELEASE, 1, f"lock:{key}", token)
    raise LockError()
```

### 9.2 Reentrant (재진입 — 같은 owner 가 여러 번)

token 대신 `owner:count` hash, 재진입 시 count++.
복잡 — Redisson 같은 라이브러리 권장.

### 9.3 Read/Write 락
Redisson 의 `RReadWriteLock` 또는 직접 구현.

### 9.4 Semaphore (N 동시)

```
SET semaphore:Q 5      # 5 permit
DECR semaphore:Q       # acquire
INCR semaphore:Q       # release
```

⚠️ overflow / negative 보호 필요 → Lua.

---

## 10. Rate Limit 도 비슷

분산 락 패턴의 형제 — 분당 N 회 제한:

```python
key = f"rate:ip:{ip}:{minute}"
count = r.incr(key)
if count == 1:
    r.expire(key, 60)
if count > LIMIT:
    raise TooManyRequests()
```

또는 sliding window — Sorted Set 의 ZRANGEBYSCORE 로 구현.

---

## 11. 실전 — Cron 싱글톤

```python
def run_cron():
    if r.set("cron:cleanup", "1", nx=True, ex=3600):
        try:
            cleanup_jobs()
        finally:
            r.delete("cron:cleanup")
```

여러 인스턴스가 같이 cron 돌려도 1 회만 실행.

---

## 12. 함정

### 함정 1 — `SETNX` + `EXPIRE` 분리
원자성 X. SETNX 후 EXPIRE 전 crash → deadlock.

### 함정 2 — token 검증 없이 DEL
다른 워커의 락 해제. 항상 Lua + token 비교.

### 함정 3 — TTL 너무 짧음
작업 끝나기 전 만료 → 동시 실행. P99 보다 크게.

### 함정 4 — TTL 너무 김
워커 죽으면 오래 lockout.

### 함정 5 — Redlock 만 믿음
강한 보장 필요 → 합의 기반 + fencing token.

### 함정 6 — clock 의존
TTL = wall clock. NTP 보정 / clock drift 주의.

### 함정 7 — 락 안에서 외부 호출
외부 API hang → TTL 만료 → 동시 실행. timeout 짧게 + retry.

### 함정 8 — 키 충돌
`lock:user:1` 과 `cache:user:1` 분리. 네이밍 규칙.

---

## 13. 학습 자료

- **Redis Lock Pattern** — redis.io/docs/manual/patterns/distributed-locks
- **Martin Kleppmann — How to do distributed locking**
- **Antirez — Is Redlock safe?**
- **Redisson Documentation** (Java)

---

## 14. 관련

- [[commands]] — SET / EVAL
- [[transactions-lua]] — Lua
- [[redis]] — Redis hub
