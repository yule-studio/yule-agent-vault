---
title: "Redis 복제 / Sentinel / Cluster"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:05:00+09:00
tags:
  - database
  - redis
  - replication
  - cluster
  - sentinel
---

# Redis 복제 / Sentinel / Cluster

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Replication / Sentinel / Cluster |

**[[redis|↑ Redis hub]]**

---

## 1. 스케일링 패턴

| 패턴 | 가용성 | 확장성 |
| --- | --- | --- |
| **Single** | ❌ | RAM 한계 |
| **Replica** | 읽기 분산 | 쓰기 X |
| **Sentinel** | 자동 failover | 쓰기 X |
| **Cluster** | 샤딩 + failover | 쓰기 + 읽기 |

---

## 2. Replica (Master-Replica)

### 2.1 설정

```ini
# replica
replicaof primary-host 6379
masterauth <password>
replica-read-only yes
```

또는 런타임:

```
REPLICAOF primary-host 6379
REPLICAOF NO ONE                  # primary 로 승격
```

### 2.2 복제 방식

- **PSYNC** — partial resync (백로그 안에 있으면)
- **Full resync** — RDB 보내고 시작
- **Diskless** — RDB 를 디스크 X, socket 으로 직접

```ini
repl-diskless-sync yes
repl-backlog-size 256mb
repl-timeout 60
```

### 2.3 상태

```
INFO replication
```

| 필드 | 의미 |
| --- | --- |
| `role` | master / slave |
| `connected_slaves` | replica 수 |
| `master_link_status` | up / down |
| `master_repl_offset` | primary 의 복제 오프셋 |
| `slave_repl_offset` | replica 의 |
| `master_link_down_since_seconds` | 끊긴 시간 |

---

## 3. Sentinel — 자동 Failover

### 3.1 구조

```
   Primary
    ↓ replica
  Replica × N

  Sentinel × M (홀수, 보통 3)
  Sentinel 들이 서로 + Redis 들을 감시
```

### 3.2 설정 — sentinel.conf

```ini
port 26379
sentinel monitor mymaster 10.0.0.1 6379 2    # 정족수 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster <password>
```

### 3.3 동작

```
1. Sentinel 들이 SDOWN 감지 (자신 기준)
2. ODOWN — 정족수 도달 (다른 Sentinel 들과 동의)
3. Leader 선출 (Raft 비슷)
4. Replica 중 가장 최신 / 우선순위 → PROMOTE
5. 다른 replica 가 새 primary 를 따르도록 CONFIG
6. 클라이언트는 Sentinel 에 질의 → 새 primary 발견
```

### 3.4 클라이언트

```
# Sentinel-aware client
redis-sentinel://sent1:26379,sent2:26379,sent3:26379/mymaster
```

라이브러리가 Sentinel 에 질의 → primary 의 주소를 받음 + 변경 시 reconnect.

### 3.5 한계
- 단일 primary = 쓰기 확장 X
- failover 동안 쓰기 짧게 중단

---

## 4. Cluster — 샤딩 + Failover

### 4.1 구조

```
16384 hash slots
각 slot → 한 primary 가 담당
각 primary → 1+ replica

CRC16(key) mod 16384 → slot
```

### 4.2 클러스터 모드 설정

```ini
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-require-full-coverage no
cluster-allow-reads-when-down no
```

### 4.3 클러스터 만들기

```bash
redis-cli --cluster create \
  10.0.0.1:6379 10.0.0.2:6379 10.0.0.3:6379 \
  10.0.0.4:6379 10.0.0.5:6379 10.0.0.6:6379 \
  --cluster-replicas 1
# → 3 primary + 3 replica
```

### 4.4 명령

```
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
CLUSTER SHARDS                    # 7.0+
CLUSTER KEYSLOT mykey
CLUSTER COUNTKEYSINSLOT 0
CLUSTER GETKEYSINSLOT 0 100
CLUSTER MEET ip port
CLUSTER FORGET nodeid
CLUSTER FAILOVER                  # 수동 failover
CLUSTER REPLICATE primaryid
```

### 4.5 redis-cli --cluster

```bash
redis-cli --cluster info host:port
redis-cli --cluster check host:port
redis-cli --cluster reshard host:port
redis-cli --cluster rebalance host:port
redis-cli --cluster add-node new:6379 host:6379
redis-cli --cluster del-node host:6379 <node-id>
```

### 4.6 MOVED / ASK

```
GET mykey
(error) MOVED 1234 10.0.0.2:6379

→ 다른 슬롯 노드로 가야 함
```

Cluster-aware 클라이언트가 자동 재시도.

`ASK` = resharding 중 임시 redirect.

### 4.7 Multi-key + Hash Tag

```
{user:1234}:profile
{user:1234}:settings
→ 같은 slot 보장 → MGET / 트랜잭션 가능
```

### 4.8 Cluster 의 한계
- 다중 키 명령은 같은 slot 만
- Lua / Transaction 도 같은 slot
- Pub/Sub 전 노드 broadcast (7.0 Sharded Pub/Sub 으로 해결)
- 다중 DB (DB 0~15) 미지원 → DB 0 만

---

## 5. 매니지드 / 변형

| 서비스 | 특징 |
| --- | --- |
| AWS ElastiCache | Sentinel / Cluster |
| AWS MemoryDB | Multi-AZ durable Cluster |
| GCP Memorystore | Standard + High Availability |
| Redis Cloud | Redis Inc. |
| Upstash | 서버리스, 글로벌 |
| Dragonfly | Redis 호환 + 멀티 스레드 |
| KeyDB | 멀티 스레드 fork |
| Valkey | Linux Foundation fork |

---

## 6. 클라이언트 종류

| 종류 | 사용처 |
| --- | --- |
| 단순 client (jedis, redis-py, ioredis) | Standalone |
| Sentinel client | Sentinel |
| Cluster client | Cluster (MOVED 자동 처리) |
| Smart client | Cluster + slot 캐시 |

대부분 라이브러리가 모드 지원.

---

## 7. Replica 의 일관성

- **Async** 가 기본 → primary commit 후 replica 가 지연된 상태
- `WAIT N timeout` — N 개 replica ack 까지 대기 (semi-sync 비슷)
- 실패해도 데이터 손실 가능 → MemoryDB / Raft 기반 (Redis Raft 모듈)

```
WAIT 1 1000        # 1 replica ack 까지 최대 1초 대기
```

---

## 8. Cluster vs Sentinel 결정

| 시나리오 | 추천 |
| --- | --- |
| 작은 워크로드, HA | Sentinel |
| 큰 워크로드 / RAM 한계 | Cluster |
| 단순 (캐시) | Sentinel 또는 ElastiCache 노드 |
| 글로벌 다중 리전 | 매니지드 (ElastiCache, Redis Cloud) |

---

## 9. 모니터링

```
INFO replication
CLUSTER INFO
CLUSTER NODES
SENTINEL master mymaster

# 라우팅 / failover 이벤트
CLIENT NO-EVICT
DEBUG SLEEP 0
```

도구:
- **RedisInsight** — GUI
- **Prometheus redis_exporter**
- **Datadog / NewRelic**

---

## 10. 함정

### 함정 1 — Replica 에 쓰기
`replica-read-only no` 위험. split-brain 가능.

### 함정 2 — Sentinel 정족수 잘못
N=2 sentinel + quorum=1 → split-brain.

### 함정 3 — Cluster `cluster-require-full-coverage = yes`
한 슬롯 down 시 클러스터 전체 read X. `= no` 권장 (downed slot 만 X).

### 함정 4 — 다중 키 명령 + Cluster
CROSSSLOT 에러. hash tag.

### 함정 5 — DB 0 외 사용
Cluster 미지원. 멀티 DB 안티 패턴.

### 함정 6 — Pub/Sub + Cluster (옛)
모든 노드 broadcast — 7.0 Sharded Pub/Sub 사용.

### 함정 7 — Failover 후 데이터 손실
async → 마지막 변경 손실 가능. `WAIT` / MemoryDB.

### 함정 8 — Slot 재분배 중 클라이언트 캐시
slot map 변경 → MOVED. Smart client.

---

## 11. 학습 자료

- **Redis Replication** — redis.io/docs/management/replication
- **Redis Sentinel** — redis.io/docs/management/sentinel
- **Redis Cluster Spec** — redis.io/docs/reference/cluster-spec
- **Designing Data-Intensive Applications** Ch. 5-6

---

## 12. 관련

- [[configuration]] — replication / cluster 설정
- [[persistence]] — replica 에서 persistence
- [[redis]] — Redis hub
