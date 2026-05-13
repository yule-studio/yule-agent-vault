---
title: "Redis 트랜잭션 / Lua / Functions"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:00:00+09:00
tags:
  - database
  - redis
  - transaction
  - lua
---

# Redis 트랜잭션 / Lua / Functions

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | MULTI/EXEC / WATCH / Lua / Functions |

**[[redis|↑ Redis hub]]**

---

## 1. MULTI / EXEC

```
MULTI
SET a 1
INCR counter
LPUSH q "task"
EXEC                              # 또는 DISCARD
```

### 1.1 보장

- **원자 실행** — 다른 명령 끼어들기 X
- **개별 명령 에러는 다른 명령 계속 실행** — 일반 DB 의 rollback X
- 큐 단계 syntax error 면 EXEC 전체 실패

⚠️ Redis 트랜잭션은 **SQL 트랜잭션과 다름** — partial 실패 가능. rollback 없음.

---

## 2. WATCH — 낙관적 락

```
WATCH balance:1

current = GET balance:1
new = current - 100

MULTI
SET balance:1 new
EXEC
# EXEC nil 이면 다른 트랜잭션이 balance:1 변경 → 재시도
```

### 2.1 패턴

```python
while True:
    pipe = r.pipeline()
    pipe.watch("balance:1")
    current = int(pipe.get("balance:1"))
    new = current - 100
    if new < 0:
        pipe.unwatch()
        raise InsufficientFunds()
    pipe.multi()
    pipe.set("balance:1", new)
    try:
        pipe.execute()
        break
    except WatchError:
        continue   # 재시도
```

---

## 3. Lua Script — 원자 + 복잡 로직

```
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
EVAL "local v = redis.call('GET', KEYS[1]); return v" 1 mykey

# 스크립트 로드 후 SHA 로 호출
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "a1b2c3..."
EVALSHA a1b2c3... 1 mykey
```

### 3.1 KEYS vs ARGV

```
EVAL script numkeys key1 key2 ... arg1 arg2 ...

KEYS[1], KEYS[2], ...
ARGV[1], ARGV[2], ...
```

**Cluster 호환** 위해 모든 키는 KEYS 로 전달. 같은 slot 보장 (hash tag).

### 3.2 redis.call vs pcall

```lua
local ok, err = pcall(redis.call, 'GET', KEYS[1])
-- 에러 시 ok=false, err=msg
```

`redis.call` 은 에러 시 스크립트 abort. `pcall` 은 캡처.

### 3.3 자주 쓰는 Lua

#### 안전 락 해제

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

#### Compare-And-Set

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('SET', KEYS[1], ARGV[2])
else
  return nil
end
```

#### Rate Limit (sliding window)

```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)
local count = redis.call('ZCARD', key)
if count >= limit then
  return 0
end
redis.call('ZADD', key, now, now)
redis.call('PEXPIRE', key, window * 1000)
return 1
```

### 3.4 Lua 제약

- **결정적이어야 함** — `math.random` 없음 (또는 `redis.replicate_commands` 후 사용 7.0 이전)
- **시간 함수 제약** — `redis.call('TIME')`
- **5 초 제한** (`lua-time-limit`) — 넘으면 SCRIPT KILL
- **single-threaded** — 다른 명령 blocking

→ **짧고 결정적** 으로.

---

## 4. Functions (7.0+) — Lua 의 진화

```lua
-- mylib.lua
#!lua name=mylib

redis.register_function('myadd', function(keys, args)
  return redis.call('INCRBY', keys[1], args[1])
end)
```

```
FUNCTION LOAD "$(cat mylib.lua)"
FCALL myadd 1 counter 5
FCALL_RO myadd 1 counter 5         # 읽기 전용 보장
FUNCTION LIST
FUNCTION DELETE mylib
```

### 4.1 장점 (vs EVAL)

- **영속** — RDB / AOF 와 함께 복제
- **버전 / 라이브러리** — 코드 관리
- **read-only 함수** 명시
- 노드 재시작 후에도 살아있음

### 4.2 라이브러리 빌드

```
FUNCTION LOAD REPLACE "..."
FUNCTION DUMP                       # 백업
FUNCTION RESTORE <dump>
FUNCTION STATS
```

---

## 5. MULTI/EXEC vs Lua vs Functions

| 항목 | MULTI/EXEC | Lua | Functions |
| --- | --- | --- | --- |
| 원자성 | ✅ | ✅ | ✅ |
| 조건 분기 | ❌ | ✅ | ✅ |
| 재사용 | ❌ | SCRIPT LOAD 후 SHA | 이름으로 |
| 영속 / 복제 | 명령 자체 | RDB 에는 X | ✅ RDB / 복제 |
| 추천 | 단순 batching | 임시 로직 | 영구 로직 |

---

## 6. Pipeline 과의 차이

```
Pipeline:
  - 클라이언트가 명령을 한 번에 전송 (round-trip 1)
  - 원자성 X — 다른 명령 사이에 끼어들 수 있음

MULTI/EXEC:
  - 큐잉 후 일괄 실행 — 사이에 다른 명령 X
  - 원자성 ✅

Lua:
  - 한 명령처럼 → 사이에 다른 명령 X
  - 원자성 + 조건 / 반복
```

Pipeline + MULTI/EXEC 같이 사용 가능 (성능 + 원자).

---

## 7. CLIENT PAUSE — 단기 동결

```
CLIENT PAUSE 1000              # 1초 동안 새 명령 거부
```

블루-그린 배포 / failover 시 사용.

---

## 8. 함정

### 함정 1 — Lua 안의 비결정적 코드
복제 / replay 시 결과 다름. 결정적으로.

### 함정 2 — Lua 길게
다른 명령 blocking. 짧게 + lua-time-limit.

### 함정 3 — MULTI/EXEC 의 rollback 기대
실패한 명령도 commit. 검증은 응용에서.

### 함정 4 — WATCH 후 같은 트랜잭션 안에서 변경
WATCH 는 다른 클라이언트의 변경 감지. 자기 변경엔 무관.

### 함정 5 — Cluster + 다중 키 Lua
모든 키 같은 slot 이어야. hash tag.

### 함정 6 — SCRIPT FLUSH 후 EVALSHA 실패
NOSCRIPT 에러. EVALSHA 실패 시 EVAL 로 fallback.

### 함정 7 — Functions 의 클러스터 전파
모든 노드에 LOAD 필요할 수 있음. 운영 도구 사용.

---

## 9. 학습 자료

- **Redis Transactions** — redis.io/docs/interact/transactions
- **Redis Lua / Functions** — redis.io/docs/interact/programmability
- **Redis Best Practices** — 라이브러리 패턴

---

## 10. 관련

- [[commands]] — MULTI / EVAL / FCALL
- [[distributed-lock]] — Lua 의 표준 사용처
- [[redis]] — Redis hub
