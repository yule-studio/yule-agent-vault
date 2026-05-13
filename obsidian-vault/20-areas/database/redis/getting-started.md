---
title: "Redis 시작하기 — 설치 / redis-cli / 첫 명령"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:25:00+09:00
tags:
  - database
  - redis
  - setup
---

# Redis 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / 첫 명령 |

**[[redis|↑ Redis hub]]**

---

## 1. 설치

### 1.1 macOS

```bash
brew install redis
brew services start redis

# 또는 일회성
redis-server
```

### 1.2 Ubuntu

```bash
sudo apt install redis-server
sudo systemctl enable --now redis-server

# 더 최신 버전
sudo add-apt-repository ppa:redislabs/redis
```

### 1.3 Docker

```bash
docker run -d \
  --name redis7 \
  -p 6379:6379 \
  -v redis7-data:/data \
  redis:7.4 redis-server --appendonly yes
```

### 1.4 매니지드

| 서비스 | 특징 |
| --- | --- |
| AWS ElastiCache | Redis / Valkey |
| AWS MemoryDB | Multi-AZ durable Redis |
| GCP Memorystore | Redis |
| Upstash | 서버리스 Redis |
| Redis Cloud | Redis Inc. 직접 |
| Valkey on cloud | 새 표준 (2024+) |

### 1.5 Valkey

```bash
brew install valkey
# 또는
docker run -d valkey/valkey:8
```

Redis 와 거의 완전 호환. 신규는 Valkey 검토.

---

## 2. redis-cli 첫 연결

```bash
redis-cli                          # 로컬 6379
redis-cli -h host -p 6379 -a pass
redis-cli -u redis://:pass@host:6379/0

# DB 0~15 (멀티 DB, 권장 X — cluster 호환)
redis-cli -n 1
```

---

## 3. 첫 명령

```
SET mykey "hello"
GET mykey

DEL mykey
EXISTS mykey

KEYS *               # ⚠️ 운영 X — blocking
SCAN 0 MATCH user:* COUNT 100   # ✅ 권장

INCR counter
EXPIRE mykey 60       # 60초 후 삭제
TTL mykey

PING                  # → PONG
INFO                  # 서버 정보
CONFIG GET maxmemory
DBSIZE                # 키 개수
FLUSHDB               # ⚠️ 현재 DB 모두 지움
FLUSHALL              # ⚠️ 모든 DB 모두 지움
```

---

## 4. 자료구조 첫 맛

```
# String
SET user:1:name "Alice"
GET user:1:name

# List (큐)
LPUSH queue "task1"
LPUSH queue "task2"
RPOP queue

# Hash (오브젝트)
HSET user:1 name "Alice" age 30
HGET user:1 name
HGETALL user:1

# Set (중복 X)
SADD tags "redis" "db" "kv"
SMEMBERS tags

# Sorted Set (점수 정렬)
ZADD leaderboard 100 "alice" 200 "bob"
ZRANGE leaderboard 0 -1 WITHSCORES
ZREVRANGE leaderboard 0 2

# Stream
XADD events * type "login" user "alice"
XREAD COUNT 10 STREAMS events 0

# Bitmap
SETBIT visits:2026-05-13 1234 1
BITCOUNT visits:2026-05-13

# HyperLogLog (대략 카운트)
PFADD uniques "user1"
PFCOUNT uniques

# Geo
GEOADD places 127.0 37.5 "seoul"
GEOSEARCH places FROMMEMBER seoul BYRADIUS 100 km
```

자세히 → [[data-types]]

---

## 5. 패스워드 / ACL

### 5.1 기본 password (옛)

```ini
# redis.conf
requirepass secret
```

```bash
redis-cli -a secret
# 또는 접속 후
AUTH secret
```

### 5.2 ACL (6.0+)

```
ACL SETUSER alice on >secret ~app:* +get +set
ACL LIST
ACL WHOAMI
```

| 패턴 | 의미 |
| --- | --- |
| `on/off` | 활성 |
| `>password` | 비번 설정 |
| `~prefix:*` | 키 접근 제한 |
| `+cmd` / `-cmd` | 명령 허용/금지 |
| `&channel` | pub-sub 채널 |

---

## 6. 연결 문자열

```
redis://[user[:password]]@host[:port][/database][?option=value]
rediss://...                          # TLS
redis-sentinel://...
```

예:
```
redis://localhost:6379/0
redis://:secret@localhost:6379
rediss://user:pass@cache.example.com:6380/0
```

---

## 7. 자주 보는 INFO 섹션

```
INFO server          # 버전, OS, 업타임
INFO clients         # 연결 수
INFO memory          # 사용량 / 단편화
INFO persistence     # RDB / AOF 상태
INFO stats           # 명령 처리량
INFO replication     # 복제 상태
INFO commandstats    # 명령별 통계
INFO keyspace        # DB 별 키 수
```

---

## 8. 함정

### 함정 1 — `KEYS *` 운영 사용
blocking. 큰 DB 면 수초 멈춤. **`SCAN`** 사용.

### 함정 2 — `FLUSHALL` 사고
ACL 로 막거나 별도 키워드.

### 함정 3 — `EXPIRE` 단위
초 단위. 밀리초는 `PEXPIRE`.

### 함정 4 — 멀티 DB (0-15) 의존
Cluster 환경에서 작동 X. 신규는 단일 DB + 키 prefix.

### 함정 5 — 비밀번호 없이 외부 노출
인터넷 직노출 = 즉시 침해. `bind 127.0.0.1` + ACL + TLS.

### 함정 6 — `appendonly no` 데이터 손실
RDB 만으로는 RPO ≥ 분 단위. AOF 권장.

---

## 9. 관련

- [[configuration]] — redis.conf
- [[data-types]] — 자료구조
- [[commands]] — 명령 사전
- [[redis]] — Redis hub
