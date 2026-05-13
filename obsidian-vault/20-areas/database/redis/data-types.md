---
title: "Redis 자료구조 — String / List / Hash / Set / ZSet / Stream / Bitmap / HLL / Geo"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:35:00+09:00
tags:
  - database
  - redis
  - data-types
---

# Redis 자료구조

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 9 가지 자료구조 + 명령 |

**[[redis|↑ Redis hub]]**

---

## 1. 한 줄

Redis 는 **자료구조의 서버**. 단순 KV 가 아니라 List / Hash / Set / ZSet / Stream / Bitmap / HLL / Geo 를 **원자 명령** 으로 다룸.

---

## 2. String

가장 기본. 최대 512 MB.

```
SET key "value"
SET key "value" EX 60        # 60초 TTL
SET key "value" NX           # 없을 때만 (락 패턴)
SET key "value" XX           # 있을 때만
GET key
DEL key
APPEND key " more"
STRLEN key

# 숫자
SET counter 0
INCR counter
INCRBY counter 5
DECRBY counter 2
INCRBYFLOAT score 1.5

# 다중
MSET a 1 b 2 c 3
MGET a b c
```

사용처: 캐시 / 토큰 / 카운터 / 임시 값.

---

## 3. List (Linked List + ziplist)

양쪽 끝 push/pop O(1). 큐 / 스택 / 작업 리스트.

```
LPUSH q "task1" "task2"          # 앞에서
RPUSH q "task3"                  # 뒤에서
LPOP q
RPOP q
LRANGE q 0 -1                    # 모두
LLEN q
LREM q 1 "task1"                 # 첫 매칭 제거
LINDEX q 0
LSET q 0 "new"
LINSERT q BEFORE "task2" "new"

# Blocking (워커가 기다림)
BLPOP q 5                        # 5초 또는 즉시
BRPOP q 0                        # 무한 대기

# Atomic 이동 (작업 큐 패턴)
LMOVE q processing LEFT RIGHT
BLMOVE q processing LEFT RIGHT 5
```

사용처: 작업 큐, 메시지 큐, 최근 N 개 (capped list).

---

## 4. Hash

키 안의 필드-값 맵 (오브젝트).

```
HSET user:1 name "Alice" age 30 email "a@x.com"
HGET user:1 name
HMGET user:1 name age
HGETALL user:1
HEXISTS user:1 name
HDEL user:1 email
HKEYS user:1
HVALS user:1
HLEN user:1
HINCRBY user:1 age 1

# Hash 안의 필드 TTL (7.4+)
HEXPIRE user:1 60 FIELDS 1 email
HTTL user:1 FIELDS 1 email
```

사용처: 사용자 / 세션 / 설정 오브젝트. 작은 hash 는 매우 메모리 효율적 (ziplist).

---

## 5. Set

중복 없는 unordered 컬렉션.

```
SADD tags "redis" "db" "kv"
SREM tags "kv"
SMEMBERS tags
SISMEMBER tags "redis"
SCARD tags                       # 개수
SRANDMEMBER tags 2               # 랜덤 N
SPOP tags                        # 랜덤 제거

# 집합 연산
SADD a 1 2 3
SADD b 2 3 4
SUNION a b                       # {1,2,3,4}
SINTER a b                       # {2,3}
SDIFF a b                        # {1}
SUNIONSTORE result a b           # 결과 저장
```

사용처: 태그 / 친구 / 유니크 카운트 / 관계.

---

## 6. Sorted Set (ZSet, Skip List + Hash)

점수로 정렬되는 unique 컬렉션.

```
ZADD lb 100 "alice" 200 "bob" 150 "carol"
ZADD lb GT 250 "alice"           # 더 큰 경우만 update (6.2+)
ZADD lb XX 250 "alice"           # 있을 때만
ZADD lb NX 50 "dave"             # 없을 때만

ZINCRBY lb 10 "alice"
ZSCORE lb "alice"
ZRANK lb "alice"                 # 오름차 순위 (0-based)
ZREVRANK lb "alice"              # 내림차 순위

# 범위 조회
ZRANGE lb 0 -1 WITHSCORES        # 오름차 모두
ZREVRANGE lb 0 9 WITHSCORES      # 상위 10
ZRANGEBYSCORE lb 100 200         # 점수 범위
ZRANGEBYLEX lb "[a" "[c"         # 사전식 범위 (같은 점수일 때)

# 7.0+ 통합 명령
ZRANGE lb 0 -1 REV WITHSCORES

# 집합 연산
ZUNIONSTORE out 2 a b WEIGHTS 1 2 AGGREGATE SUM
ZINTERSTORE
ZDIFFSTORE

# 제거
ZREM lb "alice"
ZREMRANGEBYRANK lb 0 -101        # 상위 100 만 남기기
ZREMRANGEBYSCORE lb -inf 50
```

사용처: leaderboard, priority queue, 시계열 인덱스, rate limit (sliding window).

---

## 7. Stream (5.0+)

Append-only 로그. Kafka 와 비슷.

```
# 추가 — * = 자동 ID
XADD events * type "login" user "alice"
XADD events MAXLEN ~ 1000 * type "view" url "/home"

# 읽기
XRANGE events - +                # 모두
XRANGE events 1700000000-0 +
XREVRANGE events + -
XLEN events

# 실시간 구독 (consumer 없이)
XREAD COUNT 10 BLOCK 0 STREAMS events $

# Consumer Group
XGROUP CREATE events workers $ MKSTREAM
XREADGROUP GROUP workers worker-1 COUNT 10 BLOCK 0 STREAMS events >
XACK events workers <id>
XPENDING events workers           # 미-ACK
XCLAIM events workers worker-2 60000 <id>   # 다른 워커가 가져감

# 트림
XTRIM events MAXLEN 1000
XTRIM events MINID 1700000000-0
```

사용처: 이벤트 로그, 작업 큐 (Pub/Sub 보다 안전 — 영속), CDC.

자세히 → [[pub-sub]]

---

## 8. Bitmap

String 위의 비트 연산.

```
SETBIT visits:2026-05-13 1234 1   # user 1234 방문
GETBIT visits:2026-05-13 1234
BITCOUNT visits:2026-05-13         # 방문자 수
BITCOUNT visits:2026-05-13 0 100   # byte 범위

# 비트 연산
BITOP AND result a b
BITOP OR  result a b
BITOP XOR result a b
BITOP NOT result a

# 비트 위치 찾기
BITPOS key 1 0                     # 첫 1 의 위치
```

사용처: DAU / MAU 카운팅, A/B 테스트, presence (online/offline).

---

## 9. HyperLogLog

대략 카운팅 (error ~ 0.81%, 12 KB 고정).

```
PFADD uniques "user1" "user2" "user3"
PFCOUNT uniques
PFCOUNT uniques other_uniques      # 합집합
PFMERGE merged a b c
```

사용처: 거대 unique 카운트 (PV / UV / 거대 분산 시스템). 정확 카운트 안 됨.

---

## 10. Geo (Sorted Set 위에 구현)

```
GEOADD places 127.0 37.5 "seoul" 139.7 35.7 "tokyo"
GEOPOS places "seoul"
GEODIST places "seoul" "tokyo" km

GEOSEARCH places FROMLONLAT 127.0 37.5 BYRADIUS 1000 km ASC
GEOSEARCH places FROMMEMBER "seoul" BYBOX 1000 1000 km ASC

GEOSEARCHSTORE dest places FROMLONLAT 127.0 37.5 BYRADIUS 1000 km
```

사용처: 근처 매장 / 차량 / 사용자.

---

## 11. JSON (RedisJSON 모듈)

```
JSON.SET user:1 $ '{"name":"Alice","age":30,"tags":["a","b"]}'
JSON.GET user:1 $.name
JSON.GET user:1 $.tags[0]
JSON.SET user:1 $.age 31
JSON.ARRAPPEND user:1 $.tags "c"
JSON.DEL user:1 $.age
```

자체 Redis 코어 X — 모듈. Redis Stack 또는 Valkey-JSON.

---

## 12. Bloom / Cuckoo / Top-K / Count-Min (RedisBloom 모듈)

```
BF.RESERVE filter 0.01 1000000     # 1% 오차, 1M
BF.ADD filter "user1"
BF.EXISTS filter "user1"

CMS.INITBYDIM cms 2000 5
CMS.INCRBY cms "page1" 1
CMS.QUERY cms "page1"

TOPK.RESERVE top10 10 2000 7 0.925
TOPK.ADD top10 "alice"
TOPK.LIST top10
```

사용처: spam / 광고 중복 / 대규모 카운팅.

---

## 13. 자료구조 선택

| 패턴 | 자료구조 |
| --- | --- |
| 단순 값 / 카운터 | String |
| 큐 / 스택 / capped log | List |
| 오브젝트 / 세션 | Hash |
| 태그 / 관계 / 유니크 | Set |
| 순위 / 우선순위 / 시계열 | Sorted Set |
| 이벤트 로그 / 작업 큐 | Stream |
| presence / DAU | Bitmap |
| 거대 unique 카운트 | HyperLogLog |
| 위치 검색 | Geo |
| 중복 / 멤버십 확인 | Bloom (모듈) |

---

## 14. 키 디자인

```
prefix:type:id[:field]
user:1234
user:1234:profile
session:abc123
cache:product:42
rate:ip:192.168.0.1
lock:order:567
```

콜론 (`:`) 으로 계층 표현 — RedisInsight 등 GUI 가 트리로 렌더.

### 14.1 너무 큰 키 회피
한 hash / list 가 수만 element → 명령이 느려짐. 분할.

### 14.2 Cluster hash tag
`{tag}` 안의 부분만으로 slot 결정 → 같은 노드 보장:

```
SET {user:1234}:profile ...
SET {user:1234}:settings ...
MGET {user:1234}:profile {user:1234}:settings   # 같은 노드 OK
```

자세히 → [[replication-cluster]]

---

## 15. TTL

```
EXPIRE key 60                     # 초
PEXPIRE key 60000                 # 밀리초
EXPIREAT key <unix_seconds>
PEXPIREAT key <unix_ms>

TTL key                           # 남은 초 (-1 무한, -2 없음)
PTTL key                          # 남은 ms
PERSIST key                       # TTL 제거
```

GET / SET 등은 TTL 유지 / 제거에 주의:
- `SET` 은 TTL 제거 (기본). `KEEPTTL` 옵션으로 유지 (6.0+).

---

## 16. SCAN — 비파괴 키 순회

```
SCAN 0 MATCH user:* COUNT 100
# 반환된 cursor 로 다음 페이지
SCAN <cursor> MATCH ... COUNT ...

HSCAN, SSCAN, ZSCAN   # 큰 hash/set/zset 순회
```

`KEYS *` 대신 항상 SCAN.

---

## 17. 명령 그룹

`@read`, `@write`, `@admin`, `@dangerous`, `@scripting`, `@string`, `@list`, ... — ACL 에 사용.

```
COMMAND LIST FILTERBY MODULE redisjson
COMMAND INFO SET
COMMAND COUNT
```

---

## 18. 함정

### 함정 1 — Big Key
한 키가 GB 단위 / 수십만 element → DEL 도 blocking. **분할** + `UNLINK` (async).

### 함정 2 — Hot Key
하나의 키에 trafffic 집중 → 단일 노드 병목. 분할 / 캐시 layer.

### 함정 3 — `KEYS *`
운영 차단 권장.

### 함정 4 — Cluster + multi-key 명령
서로 다른 slot 의 키를 한 명령에 → CROSSSLOT 에러. hash tag.

### 함정 5 — Type mismatch
같은 키에 String 저장 후 LPUSH → WRONGTYPE.

### 함정 6 — `SET` 의 기본 TTL 제거
기존 TTL 사라짐. `KEEPTTL` 옵션.

### 함정 7 — HyperLogLog 의 정확도 기대
1% 오차 — 절대 정확 카운트 X.

---

## 19. 학습 자료

- **Redis Commands Reference** — redis.io/commands
- **Redis Data Structures** — university.redis.com
- **antirez 블로그** — 자료구조 설계 통찰

---

## 20. 관련

- [[commands]] — 명령 사전
- [[pub-sub]] — pub/sub & streams
- [[redis]] — Redis hub
