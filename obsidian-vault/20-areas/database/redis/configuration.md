---
title: "Redis 설정 — redis.conf / maxmemory / persistence"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:30:00+09:00
tags:
  - database
  - redis
  - configuration
---

# Redis 설정 — redis.conf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 핵심 옵션 |

**[[redis|↑ Redis hub]]**

---

## 1. 설정 파일

```
/etc/redis/redis.conf           # 운영
~/.config/redis/redis.conf
```

### 1.1 적용 방법

```bash
redis-server /etc/redis/redis.conf
```

런타임 변경:
```
CONFIG GET maxmemory
CONFIG SET maxmemory 4gb
CONFIG REWRITE          # 파일에 영구화
```

---

## 2. 네트워크

```ini
bind 0.0.0.0 -::*       # 모든 인터페이스 (보안 주의)
bind 127.0.0.1 -::1     # 로컬만

protected-mode yes       # bind / password 없으면 localhost 만
port 6379
tcp-backlog 511
timeout 0                # idle disconnect — 0 = 무제한
tcp-keepalive 300
```

---

## 3. 일반

```ini
daemonize no
supervised systemd
pidfile /var/run/redis/redis.pid
loglevel notice          # debug, verbose, notice, warning
logfile /var/log/redis/redis.log
databases 16             # cluster 모드에선 0 만 사용
```

---

## 4. 메모리

### 4.1 maxmemory (가장 중요)

```ini
maxmemory 4gb
```

설정 안 하면 무한 → OS OOM 위험. **반드시 설정**.

### 4.2 maxmemory-policy

```ini
maxmemory-policy allkeys-lru
```

| 정책 | 의미 |
| --- | --- |
| `noeviction` | 메모리 부족 시 write 에러 |
| `allkeys-lru` | 모든 키 중 LRU 제거 |
| `volatile-lru` | TTL 있는 키 중 LRU |
| `allkeys-lfu` | 모든 키 중 LFU (4.0+) |
| `volatile-lfu` | TTL 있는 키 중 LFU |
| `allkeys-random` | 랜덤 |
| `volatile-random` | TTL 있는 키 랜덤 |
| `volatile-ttl` | TTL 짧은 것 우선 |

캐시 = `allkeys-lru` 또는 `allkeys-lfu`.
영속 KV = `noeviction`.

### 4.3 maxmemory-samples

```ini
maxmemory-samples 5      # LRU/LFU 샘플 수
```

5 → 10 으로 늘리면 정확 ↑, CPU ↑.

---

## 5. Persistence

### 5.1 RDB Snapshot

```ini
save 3600 1              # 1 시간에 1 키 변경 → snapshot
save 300 100             # 5분에 100 변경
save 60 10000            # 1분에 1만 변경

dbfilename dump.rdb
dir /var/lib/redis
rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes
```

비활성:
```ini
save ""
```

### 5.2 AOF (Append Only File)

```ini
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec     # always / everysec / no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

| fsync | 동작 | Durability |
| --- | --- | --- |
| `always` | 매 명령 fsync | ★★★ 가장 안전, 느림 |
| `everysec` (기본) | 1초마다 | ★★ 1초 손실 가능 |
| `no` | OS 에 맡김 | ★ 빠름, 위험 |

자세히 → [[persistence]]

---

## 6. 복제

```ini
replicaof <primary_host> <primary_port>
masterauth <password>
replica-read-only yes
replica-serve-stale-data yes
repl-diskless-sync yes
repl-backlog-size 256mb
```

`replicaof no one` — Primary 로 승격.

---

## 7. 보안

### 7.1 패스워드

```ini
requirepass secret      # 옛 방식
```

### 7.2 ACL (6.0+)

```ini
user default off
user alice on >secret ~app:* +@read +@write
```

```
# /etc/redis/users.acl
user app on #SHA256HASH ~* +@all
```

```ini
aclfile /etc/redis/users.acl
```

### 7.3 TLS

```ini
tls-port 6380
port 0                   # plain port 끔
tls-cert-file /etc/redis/server.crt
tls-key-file /etc/redis/server.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes     # mTLS
```

### 7.4 명령 변경 / 비활성

```ini
rename-command FLUSHALL ""
rename-command CONFIG ""    # 또는 random suffix
rename-command DEBUG ""
```

---

## 8. 클라이언트 제한

```ini
maxclients 10000
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

slow pub/sub 구독자가 buffer 폭증 → 자동 끊기.

---

## 9. Slow Log

```ini
slowlog-log-slower-than 10000    # μs (10 ms)
slowlog-max-len 128
```

```
SLOWLOG GET 10
SLOWLOG RESET
```

---

## 10. Latency Monitor (6.0+)

```ini
latency-monitor-threshold 100   # ms
```

```
LATENCY LATEST
LATENCY HISTORY <event>
LATENCY GRAPH <event>
```

---

## 11. 메모리 단편화

```
INFO memory
# mem_fragmentation_ratio
```

1.0 ~ 1.5 OK. 큰 값은 단편화.

```ini
activedefrag yes
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
```

---

## 12. THP / Kernel

```bash
# THP 끔 (권장)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Memory overcommit
echo 1 > /proc/sys/vm/overcommit_memory

# ulimit
ulimit -n 65535
```

`overcommit_memory = 1` 이면 BGSAVE / BGREWRITEAOF 의 fork 가 안전.

---

## 13. 권장 시작 redis.conf — 8GB 캐시

```ini
bind 127.0.0.1 -::1
port 6379
protected-mode yes

maxmemory 6gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# 캐시면 persistence off (또는 가벼운 RDB 만)
save ""
appendonly no

# 영속 KV 라면
# save 3600 1
# appendonly yes
# appendfsync everysec

loglevel notice
logfile /var/log/redis/redis.log

slowlog-log-slower-than 10000
slowlog-max-len 256
latency-monitor-threshold 100

# 보안
requirepass <strong>
rename-command FLUSHALL ""
rename-command CONFIG ""
```

---

## 14. 함정

### 함정 1 — `maxmemory` 미설정
무한 사용 → OOM kill. 반드시 설정.

### 함정 2 — `noeviction` + 캐시
메모리 차면 write 거부. 캐시는 `allkeys-lru` / `allkeys-lfu`.

### 함정 3 — `bind 0.0.0.0` 패스워드 없이
즉시 침해. ACL + TLS + 방화벽.

### 함정 4 — `save ""` + AOF off
crash 시 완전 손실. 캐시면 OK, 영속이면 위험.

### 함정 5 — `appendfsync always`
매우 느림. 진짜 ACID 만 (드뭄).

### 함정 6 — THP on
fork 지연 / latency spike. 끄기.

### 함정 7 — `vm.overcommit_memory = 0`
큰 BGSAVE 실패. `= 1`.

### 함정 8 — `client-output-buffer-limit` 기본 너무 작음
큰 응답 끊김. 워크로드에 맞게.

---

## 15. 학습 자료

- **Redis Documentation** — Configuration / Persistence / Security
- **Redis Best Practices** — Redis Labs

---

## 16. 관련

- [[persistence]] — RDB / AOF
- [[replication-cluster]] — replica / cluster
- [[redis]] — Redis hub
