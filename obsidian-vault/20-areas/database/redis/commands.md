---
title: "Redis 명령 사전 (자주 쓰는 것 위주)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:40:00+09:00
tags:
  - database
  - redis
  - commands
---

# Redis 명령 사전

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 자주 쓰는 명령 분류 |

**[[redis|↑ Redis hub]]**

---

> 자료구조 별 명령은 [[data-types]] 에 따로. 여기는 **공통 / 관리 / 키 / 서버** 명령.

---

## 1. Key 관리

```
EXISTS key [key...]               # 존재 — 1/0
DEL key [key...]                  # 동기 삭제
UNLINK key [key...]               # 비동기 삭제 (큰 키 권장)
TYPE key                          # string / list / hash / set / zset / stream
OBJECT ENCODING key               # 내부 인코딩 (ziplist 등)
OBJECT IDLETIME key               # 마지막 접근 후 초
OBJECT FREQ key                   # LFU 빈도
RENAME old new
RENAMENX old new                  # 새 키 없을 때만
RANDOMKEY                         # 임의 키
DUMP key                          # 직렬화
RESTORE key ttl <dump>            # 역직렬화
COPY src dst [REPLACE]
TOUCH key [key...]                # idle time 갱신
```

---

## 2. TTL

```
EXPIRE key 60                     # 초
EXPIREAT key 1700000000           # 절대 시간 (unix)
PEXPIRE key 60000                 # ms
PEXPIREAT key 1700000000000

TTL key                           # 초 — -1 무한, -2 없음
PTTL key                          # ms
EXPIRETIME key                    # 절대 시간 (7.0+)
PEXPIRETIME key

PERSIST key                       # TTL 제거
```

### 2.1 옵션 (7.0+)

```
EXPIRE key 60 NX                  # TTL 없을 때만
EXPIRE key 60 XX                  # TTL 있을 때만
EXPIRE key 60 GT                  # 더 클 때만
EXPIRE key 60 LT                  # 더 작을 때만
```

---

## 3. SCAN — 순회

```
SCAN 0 MATCH user:* COUNT 100 TYPE string
HSCAN hash 0 MATCH * COUNT 100
SSCAN set 0
ZSCAN zset 0
```

cursor=0 부터 반환된 cursor 다음 페이지. cursor=0 으로 끝.

---

## 4. Database

```
SELECT 0                          # DB 0~15
DBSIZE                            # 현재 DB 키 수
FLUSHDB [ASYNC | SYNC]            # 현재 DB 비움
FLUSHALL [ASYNC | SYNC]           # 모든 DB 비움
SWAPDB 0 1
MOVE key 1                        # DB 1 로 이동
```

⚠️ 멀티 DB 는 Cluster 호환 X.

---

## 5. 서버

```
PING [msg]                        # → PONG
ECHO "hi"
INFO [section]                    # server, clients, memory, persistence, stats, replication, cpu, commandstats, keyspace
TIME                              # [seconds, microseconds]
DBSIZE
LASTSAVE                          # 마지막 RDB 시간
DEBUG SLEEP 5                     # 블로킹 시뮬레이션 (개발)

CLIENT LIST
CLIENT ID
CLIENT GETNAME / SETNAME
CLIENT KILL ID <id>
CLIENT PAUSE 1000

CONFIG GET <param>
CONFIG SET <param> <value>
CONFIG REWRITE
CONFIG RESETSTAT

LATENCY LATEST
LATENCY HISTORY <event>
LATENCY RESET

SLOWLOG GET [N]
SLOWLOG LEN
SLOWLOG RESET

MEMORY USAGE key                  # 키 메모리
MEMORY STATS
MEMORY DOCTOR
MEMORY PURGE                      # jemalloc 단편화 정리

SHUTDOWN [NOSAVE | SAVE]
```

---

## 6. 인증 / ACL

```
AUTH [user] password
ACL WHOAMI
ACL LIST
ACL CAT [@category]
ACL GETUSER alice
ACL SETUSER alice on >pass ~prefix:* +@read +@write
ACL DELUSER alice
ACL LOG                           # 인증 실패 로그
```

---

## 7. Pub/Sub

```
SUBSCRIBE channel [channel...]
UNSUBSCRIBE [channel...]
PSUBSCRIBE pattern.*
PUBLISH channel message
PUBSUB CHANNELS [pattern]
PUBSUB NUMSUB [channel...]
PUBSUB NUMPAT

# Sharded (7.0+) — Cluster 에서 효율적
SSUBSCRIBE channel
SPUBLISH channel message
```

자세히 → [[pub-sub]]

---

## 8. Stream

```
XADD stream * field value [field value ...]
XADD stream MAXLEN ~ 1000 * ...
XLEN stream
XRANGE stream - +
XREVRANGE stream + -
XREAD COUNT 10 BLOCK 0 STREAMS stream $
XREAD COUNT 10 STREAMS stream id

# Consumer Group
XGROUP CREATE stream groupname $ [MKSTREAM]
XREADGROUP GROUP group consumer COUNT 10 BLOCK 0 STREAMS stream >
XACK stream group id [id ...]
XPENDING stream group [IDLE ms] [start end count] [consumer]
XCLAIM stream group consumer min-idle id [id ...]
XAUTOCLAIM stream group consumer min-idle 0

XINFO STREAM stream
XINFO GROUPS stream
XINFO CONSUMERS stream group
XDEL stream id
XTRIM stream MAXLEN 1000
```

자세히 → [[data-types#7-stream-50]]

---

## 9. 트랜잭션 / Script

```
MULTI
SET a 1
INCR counter
EXEC
DISCARD

WATCH key [key...]                # 낙관적 락
UNWATCH

EVAL "return redis.call('GET', KEYS[1])" 1 mykey
EVALSHA <sha> 1 mykey
SCRIPT LOAD "return ..."
SCRIPT EXISTS <sha>
SCRIPT FLUSH

# Functions (7.0+)
FUNCTION LOAD "..."
FCALL fname numkeys key arg
FCALL_RO ...
FUNCTION LIST
FUNCTION DELETE fname
```

자세히 → [[transactions-lua]]

---

## 10. 복제 / Sentinel / Cluster

```
REPLICAOF host port               # 또는 SLAVEOF (옛)
REPLICAOF NO ONE                  # primary 로 승격

INFO replication

# Sentinel (별도 binary)
SENTINEL master mymaster
SENTINEL replicas mymaster
SENTINEL failover mymaster

# Cluster
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
CLUSTER SHARDS                    # 7.0+
CLUSTER KEYSLOT mykey
CLUSTER COUNTKEYSINSLOT slot
CLUSTER GETKEYSINSLOT slot count
CLUSTER RESET
CLUSTER MEET ip port
CLUSTER FORGET nodeid
CLUSTER REPLICATE primaryid
CLUSTER FAILOVER
```

자세히 → [[replication-cluster]]

---

## 11. Persistence

```
SAVE                              # blocking — 운영 X
BGSAVE                            # 비동기 RDB
LASTSAVE
BGREWRITEAOF                      # AOF 재작성
```

자세히 → [[persistence]]

---

## 12. 자주 쓰는 패턴 명령

### 12.1 카운터

```
INCR pageviews:home
INCRBY votes:42 5
```

### 12.2 Rate Limit (간단)

```
INCR rate:ip:1.2.3.4
EXPIRE rate:ip:1.2.3.4 60 NX
GET rate:ip:1.2.3.4
```

### 12.3 분산 락

```
SET lock:order:42 $token NX EX 30
# 작업 후
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:order:42 $token
```

자세히 → [[distributed-lock]]

### 12.4 Set 멤버 체크 (Bloom 대안)

```
SADD seen:emails "a@x.com"
SISMEMBER seen:emails "a@x.com"
SMISMEMBER seen:emails "a@x.com" "b@x.com"   # 다중
```

### 12.5 leaderboard

```
ZINCRBY lb 1 "alice"
ZREVRANGE lb 0 9 WITHSCORES
ZREVRANK lb "alice"
```

---

## 13. 디버그 / 분석

```
DEBUG OBJECT key
OBJECT ENCODING key
MEMORY USAGE key SAMPLES 5
OBJECT IDLETIME key
OBJECT FREQ key

CLUSTER COUNT-FAILURE-REPORTS nodeid
CLIENT LIST TYPE normal

LATENCY DOCTOR
MEMORY DOCTOR
```

---

## 14. CLI 옵션

```bash
redis-cli -h host -p 6379 -a pass
redis-cli -n 3                    # DB 3
redis-cli --bigkeys                # 큰 키 찾기
redis-cli --memkeys                # 메모리 분석
redis-cli --hotkeys                # 핫 키 (maxmemory-policy=allkeys-lfu 필요)
redis-cli --latency
redis-cli --latency-history -i 1
redis-cli --stat
redis-cli --scan --pattern 'user:*'
redis-cli --csv "ZRANGE lb 0 -1 WITHSCORES"
redis-cli --eval script.lua key1 key2 , arg1 arg2
redis-cli --rdb /tmp/dump.rdb     # streaming dump
```

---

## 15. PIPELINE — 다수 명령 일괄

```bash
echo -e "SET a 1\nSET b 2\nSET c 3" | redis-cli --pipe
```

```python
# Python
pipe = r.pipeline()
pipe.set("a", 1)
pipe.set("b", 2)
results = pipe.execute()
```

PIPELINE = round-trip 1 회. **트랜잭션 X** (atomic 보장 안 됨, 단지 batching).

---

## 16. 함정

### 함정 1 — `KEYS *` 운영 사용
blocking. SCAN.

### 함정 2 — `FLUSHALL` 사고
ACL / rename-command 로 차단.

### 함정 3 — 큰 키의 DEL
blocking. UNLINK.

### 함정 4 — PIPELINE 의 atomic 오해
PIPELINE 은 batching, 원자성 X. 원자가 필요하면 MULTI/EXEC 또는 Lua.

### 함정 5 — `WATCH` 후 SET 만
WATCH + MULTI/EXEC 한 세트. EXEC 가 실패하면 재시도.

### 함정 6 — Cluster + multi-key
CROSSSLOT. hash tag 사용.

### 함정 7 — 명령 deprecation
SLAVEOF → REPLICAOF, MASTER → PRIMARY (점진적). 새 명령 권장.

---

## 17. 학습 자료

- **redis.io/commands** — 공식 명령 사전 (검색 / 카테고리)
- **RedisInsight** — GUI
- **redis-cli --help**

---

## 18. 관련

- [[data-types]] — 자료구조별 명령
- [[transactions-lua]] — Lua / Functions
- [[distributed-lock]]
- [[redis]] — Redis hub
