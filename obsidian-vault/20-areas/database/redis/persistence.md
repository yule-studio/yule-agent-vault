---
title: "Redis Persistence — RDB / AOF / 하이브리드"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:45:00+09:00
tags:
  - database
  - redis
  - persistence
---

# Redis Persistence — RDB / AOF

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RDB / AOF / 하이브리드 |

**[[redis|↑ Redis hub]]**

---

## 1. 두 종류

| 방식 | 단위 | 복구 | 성능 |
| --- | --- | --- | --- |
| **RDB** | 시점 스냅샷 | 빠름 (단순 load) | 빠름 (background fork) |
| **AOF** | append-only 로그 | 느림 (replay) | fsync 정책에 따라 |
| **하이브리드** | RDB + AOF tail | 중간 | 중간 |

---

## 2. RDB Snapshot

### 2.1 동작

```
1. BGSAVE 또는 save 조건 만족
2. 부모가 fork() → 자식이 메모리 snapshot 을 dump.rdb 로 기록
3. 자식 종료 후 atomic rename
```

부모는 계속 요청 처리. **Copy-on-Write** 로 메모리 분리.

### 2.2 설정

```ini
save 3600 1
save 300 100
save 60 10000

dbfilename dump.rdb
dir /var/lib/redis
rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes
```

`save ""` → 자동 RDB 비활성.

### 2.3 수동

```
BGSAVE
LASTSAVE                          # 마지막 저장 시각
DEBUG RELOAD                      # 메모리 → RDB → 메모리 reload (테스트)
```

### 2.4 장단점

✅ 빠른 시작 (load 단순), 백업 운반 쉬움
❌ snapshot 사이의 변경 손실 (RPO 분 단위)

---

## 3. AOF (Append-Only File)

### 3.1 동작

```
모든 write 명령을 append-only 로 텍스트 / RESP 로 기록
→ 시작 시 replay
```

### 3.2 설정

```ini
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

auto-aof-rewrite-percentage 100   # 2 배 됐을 때 재작성
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes          # 하이브리드
```

### 3.3 fsync 정책

| 값 | 동작 | 손실 | 성능 |
| --- | --- | --- | --- |
| `always` | 매 명령 fsync | 0 | 느림 |
| `everysec` (기본) | 1 초마다 | ~1 초 | 보통 |
| `no` | OS 에 맡김 | OS 캐시 길이 | 빠름 |

### 3.4 AOF 재작성 (Rewrite)

AOF 가 크면 BGREWRITEAOF — 현재 메모리 상태를 최소 명령으로 재기록.

```
BGREWRITEAOF
```

자동: `auto-aof-rewrite-percentage` 도달 시.

### 3.5 장단점

✅ RPO 짧음 (fsync 정책에 따라), 텍스트 가독성
❌ 파일 큼, 시작 느림, 재작성 부담

---

## 4. 하이브리드 (Mixed) — 권장

```ini
aof-use-rdb-preamble yes
```

AOF rewrite 시 앞부분은 RDB 바이너리, 이후는 AOF 텍스트.
→ 빠른 load + 짧은 RPO.

---

## 5. 어떤 걸 써야 하나

| 시나리오 | 추천 |
| --- | --- |
| 단순 캐시 (RAM 충분) | persistence off |
| 캐시 + 워밍 부담 | RDB (시작 시 워밍) |
| 영속 KV / 가벼운 워크로드 | AOF everysec |
| 강한 durability | AOF everysec + replica |
| 매우 강한 (금융 등) | AOF always + sync 복제 |

⚠️ Redis 는 **주력 영속 저장소 X** 가 권장. 거대 / 중요 데이터는 RDB → DB.

---

## 6. 무손실 영속 = Redis 만으로는 어려움

```
AOF always + replica sync 도
- fork() 시 메모리 부족 (Linux overcommit_memory)
- 디스크 풀
- 단일 노드 장애
```

→ **MemoryDB** (AWS) 같은 multi-AZ durable 서비스 사용 또는 PostgreSQL.

---

## 7. fork 의 위험

### 7.1 메모리

```
fork() → CoW
부모가 모든 페이지 쓰면 → 자식도 그만큼 메모리
```

→ Redis 가 4 GB 인데 호스트 RAM 8 GB 면, write 가 많을 때 RAM 16 GB 필요할 수 있음.

```bash
echo 1 > /proc/sys/vm/overcommit_memory
```

### 7.2 latency

큰 인스턴스에서 fork 자체 시간 → 수 ms ~ 수 100 ms 멈춤.

```
LATENCY LATEST
INFO stats
# latest_fork_usec
```

→ **THP off** + 작은 인스턴스 여러 개 + diskless replication.

---

## 8. Diskless Replication

```ini
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-diskless-load disabled       # 또는 swapdb / on-empty-db (7.0+)
```

Primary 가 RDB 를 디스크 안 쓰고 바로 socket 으로 → 디스크 작은 클라우드 환경.

---

## 9. 백업

### 9.1 RDB 만 — 가장 단순

```bash
# /var/lib/redis/dump.rdb 를 정기 복사 → S3
aws s3 cp /var/lib/redis/dump.rdb s3://backups/redis/$(date +%F).rdb
```

### 9.2 BGSAVE → 복사

```bash
redis-cli BGSAVE
# wait
while [ "$(redis-cli LASTSAVE)" -le "$prev" ]; do sleep 1; done
cp /var/lib/redis/dump.rdb /backup/
```

### 9.3 redis-cli --rdb

```bash
redis-cli --rdb /backup/dump.rdb       # 원격에서도 streaming
```

### 9.4 AOF 복사
AOF 는 append-only — 진행 중에도 copy 가능 (snapshot consistency 는 아니지만 replay 가능).

---

## 10. 복원

### 10.1 RDB

```bash
systemctl stop redis
cp /backup/dump.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb
systemctl start redis
```

### 10.2 AOF

```bash
cp /backup/appendonly.aof /var/lib/redis/
systemctl start redis
# 자동 replay
```

### 10.3 AOF 손상 복구

```bash
redis-check-aof --fix /var/lib/redis/appendonly.aof
```

뒷부분 잘라냄 (마지막 손상). 약간의 손실.

---

## 11. AWS MemoryDB

Redis 와 호환되는 **multi-AZ durable** Redis. AOF 대신 분산 transactional log → 진짜 RDB.

→ Redis 가 진짜 영속 DB 가 되어야 한다면 MemoryDB.

---

## 12. 모니터링

```
INFO persistence
```

| 필드 | 의미 |
| --- | --- |
| `rdb_changes_since_last_save` | 마지막 RDB 이후 변경 |
| `rdb_bgsave_in_progress` | 현재 BGSAVE 중 |
| `rdb_last_save_time` | 마지막 RDB 시각 |
| `rdb_last_bgsave_status` | ok/err |
| `aof_enabled` | AOF on/off |
| `aof_rewrite_in_progress` | rewrite 중 |
| `aof_current_size` | 현재 크기 |
| `aof_base_size` | rewrite 직후 크기 |
| `aof_pending_rewrite` | rewrite 대기 |

---

## 13. 함정

### 함정 1 — Persistence 끈 채 운영 + crash
모두 손실. 캐시면 OK, 영속이면 위험.

### 함정 2 — `appendfsync always` 의 성능
매 명령 fsync — 수 천 TPS 한계. 의도적 선택만.

### 함정 3 — `vm.overcommit_memory = 0` + fork 실패
BGSAVE / BGREWRITEAOF 실패. `= 1`.

### 함정 4 — `stop-writes-on-bgsave-error yes` + 디스크 풀
write 거부. 알람 필수.

### 함정 5 — AOF 단편 손상 자동 복구의 손실
재시작 후 일부 명령 사라짐. 항상 백업.

### 함정 6 — Primary 의 RDB 부담 → replica 사용
Persistence 는 replica 에서 (Primary 는 write 위주).

### 함정 7 — fork latency 의 spike
THP off + 작은 메모리 + diskless replication.

---

## 14. 학습 자료

- **Redis Persistence Documentation** — redis.io/docs/management/persistence
- **AWS MemoryDB Docs**
- **antirez 글** — RDB vs AOF

---

## 15. 관련

- [[configuration]] — save / appendonly
- [[replication-cluster]] — replica 에서 persistence
- [[redis]] — Redis hub
