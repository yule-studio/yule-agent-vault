---
title: "Redis (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:20:00+09:00
tags:
  - database
  - nosql
  - kv
  - redis
  - hub
---

# Redis (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 10 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**인메모리 데이터 구조 서버 (Data Structure Server)**.
Salvatore Sanfilippo 2009 — 단순한 KV 가 아니라 **자료구조의 서버**.
μ초 응답. 캐시 / 큐 / 분산 락 / 카운터 / leaderboard / pub-sub 의 표준.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2009 | Redis 1.0 (Salvatore "antirez") |
| 2010 | VMware 인수 |
| 2013 | Redis Labs 설립 |
| 2015 | Redis Cluster (3.0) |
| 2017 | Modules (4.0) — RediSearch, RedisJSON |
| 2018 | Streams (5.0) |
| 2020 | RESP3 + ACL (6.0) |
| 2022 | Functions / Sharded pub-sub (7.0) |
| 2024 | **라이선스 변경 (RSALv2/SSPL)** — Valkey fork (Linux Foundation) |

### 2.1 Valkey
2024 라이선스 변화 후 AWS / Google / Oracle / Linux Foundation 이 **Valkey** 로 fork. 사실상 BSD Redis 의 후계자.

---

## 3. Redis 의 특징 (요약)

1. **인메모리** — μ초 latency
2. **자료구조 풍부** — String, List, Hash, Set, Sorted Set, Stream, Bitmap, HyperLogLog, Geo
3. **단일 스레드** (코어) — 동시성 문제 단순
4. **Persistence** — RDB snapshot + AOF
5. **Replication** — async, Sentinel 자동 failover
6. **Cluster** — 16384 slot sharding
7. **Modules** — Search, JSON, TimeSeries, Bloom, Graph
8. **Pub/Sub + Streams** — 메시징

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / redis-cli / 첫 명령 |
| [[configuration]] | redis.conf / maxmemory / persistence |
| [[data-types]] | String / List / Hash / Set / ZSet / Stream / Bitmap / HLL / Geo |
| [[commands]] | 가장 많이 쓰는 명령 (사전) |
| [[persistence]] | RDB / AOF / 하이브리드 |
| [[pub-sub]] | Pub/Sub / Sharded / Keyspace Notifications |
| [[distributed-lock]] | SETNX / Redlock / 만료 |
| [[transactions-lua]] | MULTI/EXEC / WATCH / Lua / Functions |
| [[replication-cluster]] | Replica / Sentinel / Cluster |
| [[security]] | requirepass / ACL / 외부 노출 사고 / 위험 명령 비활성 |
| [[use-cases]] | 언제 Redis 를 선택해야 하나 |

---

## 5. Redis 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 세션 저장 | ✅ 표준 |
| API / DB 쿼리 결과 캐시 | ✅ 표준 |
| Rate Limiting / 카운터 | ✅ |
| Leaderboard / 순위 | ✅ (Sorted Set) |
| 분산 락 | ✅ |
| 작업 큐 / 메시지 큐 | ✅ (Streams) |
| Pub/Sub 알림 | ✅ |
| Geo 검색 (가까운 매장) | ✅ |
| HyperLogLog 카운트 | ✅ |
| 주력 영속 데이터 저장 | ⚠️ 가능하지만 RDB 가 표준 |
| 풀텍스트 검색 | ⚠️ — RediSearch 또는 ES |
| 큰 데이터셋 (TB+) | ❌ — RAM 한계 |

---

## 6. 면접 핵심 질문

1. **Redis 가 빠른 이유** — 인메모리 + 단일 스레드 + I/O 멀티플렉싱.
2. **RDB vs AOF** — 장단점.
3. **분산 락 구현** — SETNX, Redlock 의 함정.
4. **Cache pattern** — cache-aside / write-through / write-behind.
5. **Cache stampede / dog-pile** 방지.
6. **Cluster** — slot, MOVED, ASK redirect.
7. **Sentinel** failover 동작.
8. **Sorted Set 의 자료구조** — Skip List + Hash.
9. **Pub/Sub vs Streams** — 차이.
10. **maxmemory-policy** — LRU / LFU / TTL.

---

## 7. 학습 자료

- **Redis Documentation** — redis.io/docs (가장 좋음)
- **Redis in Action** — Carlson
- **Designing Data-Intensive Applications** Ch. 5 (Replication)
- **Redis University** — university.redis.com (무료)
- **antirez's blog** — 옛 글, 깊은 통찰

---

## 8. 관련

- [[../mongodb/mongodb]] — Document DB
- [[../postgresql/postgresql]] — RDB (영속 표준)
- [[../database-tuning/database-tuning]] — 캐시 전략
- [[../database|↑ database hub]]
