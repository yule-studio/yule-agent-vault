---
title: "Redis — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:10:00+09:00
tags:
  - database
  - redis
  - use-cases
---

# Redis — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[redis|↑ Redis hub]]**

---

## 1. 한 줄

> "**μ초가 필요할 때** Redis".

빠른 응답 + 자료구조 풍부 = 캐시 / 큐 / 락 / 카운터 / leaderboard 의 표준.

---

## 2. ✅ 적합

### 2.1 캐시 (1번 사용처)
- API / DB 쿼리 결과
- Cache-aside / Read-through 패턴
- TTL 으로 자동 만료

### 2.2 세션 저장
- 빠른 read/write
- TTL 자연스러움
- 분산 세션 (sticky session 불필요)

### 2.3 Rate Limiting
- INCR + EXPIRE
- sliding window (Sorted Set)
- token bucket (Lua)

### 2.4 분산 락
- SET NX EX
- 짧은 작업 / 외부 API dedup
- 자세히 → [[distributed-lock]]

### 2.5 카운터 / 메트릭
- 페이지뷰 / 좋아요 / 투표
- 원자 INCR / DECR

### 2.6 Leaderboard / 순위
- Sorted Set
- 실시간 ranking

### 2.7 작업 큐
- List (LPUSH / BRPOP) — 간단
- Streams + Consumer Group — 신뢰성

### 2.8 Pub/Sub 알림
- 휘발 OK 면 채널
- 영속 / 재처리 → Streams

### 2.9 Geo 검색
- 가까운 매장 / 차량 / 사용자
- GEOSEARCH

### 2.10 Bloom / HLL — 대규모 카운팅
- HyperLogLog 거대 UV
- Bloom 중복 체크

### 2.11 Feature flag / 설정 핫 캐시
- 매 요청 DB 접근 X
- 변경 즉시 broadcast (Pub/Sub)

---

## 3. ⚠️ 검토

### 3.1 주력 영속 저장소
- RAM 한계, AOF 도 손실 가능
- **AWS MemoryDB** 라면 OK (multi-AZ durable)
- 일반적으론 PostgreSQL / MySQL 이 주력 + Redis 캐시

### 3.2 거대 데이터 (TB+)
- RAM 비싸 — Cluster 도 한계
- 콜드 데이터는 별도 (S3 / DB)

### 3.3 복잡 쿼리 / JOIN
- Redis 는 KV 중심
- RediSearch 모듈 있지만 ES 만큼은 아님

### 3.4 풀텍스트 검색 (큰 규모)
- RediSearch — 작은 규모 OK
- 큰 규모 → Elasticsearch

### 3.5 메시지 큐 (큰 규모)
- Streams OK, 거대 throughput → Kafka

---

## 4. ❌ 부적합

### 4.1 강한 ACID 필요
- 금융 거래 / 회계 — RDB

### 4.2 거대 영속 데이터셋
- TB+ → 디스크 기반 DB

### 4.3 복잡한 분석
- OLAP — ClickHouse / BigQuery

### 4.4 그래프 traversal
- 깊은 관계 → Neo4j

---

## 5. Redis vs Memcached

| 항목 | Redis | Memcached |
| --- | --- | --- |
| 자료구조 | 풍부 | 문자열만 |
| Persistence | RDB / AOF | ❌ |
| Pub/Sub | ✅ | ❌ |
| Cluster | ✅ | 클라이언트 샤딩 |
| 멀티 스레드 | 단일 (I/O 는 멀티) | 멀티 |
| 메모리 효율 | 매우 좋음 | 좋음 |
| 단순함 | 풍부 | 매우 단순 |

→ 대부분 Redis 우위. Memcached 는 단순 KV 캐시만 필요할 때.

---

## 6. Redis vs Hazelcast / Infinispan

| 항목 | Redis | Hazelcast |
| --- | --- | --- |
| 언어 | C | Java |
| 클라이언트 | 모든 언어 | Java 친화 |
| 자료구조 | KV + 풍부 | Java 컬렉션 분산 |
| 통합 | Spring / 어떤 언어 | Spring 친화 |
| 운영 | 단순 | Java cluster |

대부분 Redis 가 표준.

---

## 7. Redis vs DynamoDB / Cosmos DB

| 항목 | Redis | DynamoDB |
| --- | --- | --- |
| 저장 | 메모리 | SSD + 분산 |
| latency | μs | ms |
| 영속 | 약 | 강 |
| 확장 | Cluster | 자동 |
| 가격 | 비쌈 (RAM) | 쓰는 만큼 |

→ DynamoDB = 영속 KV 분산. Redis = 빠른 캐시 / 자료구조. **DynamoDB + DAX (Redis-like cache)** 도 흔함.

---

## 8. Redis vs Valkey vs KeyDB vs Dragonfly

| 항목 | Redis | Valkey | KeyDB | Dragonfly |
| --- | --- | --- | --- | --- |
| 라이선스 | RSALv2/SSPL | BSD | BSD | BSL |
| 호환성 | 표준 | 거의 완전 | 거의 | 거의 |
| 멀티스레드 | I/O 만 | I/O 만 | ✅ | ✅ |
| 미래 | Redis Inc. | Linux Foundation | Snap | DragonflyDB |

신규는 **Valkey** 또는 표준 Redis. KeyDB / Dragonfly 는 멀티스레드 / 성능이 핵심일 때.

---

## 9. 결정 트리

```
캐시?
└── Redis (또는 ElastiCache / Upstash)

세션?
└── Redis

분산 락?
└── Redis (짧은 임계) / Zookeeper (강한 보장)

작업 큐?
├── 단순 / 가벼움 → Redis Streams
└── 거대 / 보존 / 분산 → Kafka

leaderboard?
└── Redis Sorted Set

순간 카운터 / DAU?
└── Redis (HLL / Bitmap)

지리 검색?
└── Redis Geo (작) / PostGIS (큰)

영속 저장소?
└── ❌ Redis 단독 X — DB + Redis 캐시
```

---

## 10. 회사 사례

| 회사 | Redis 사용 |
| --- | --- |
| Twitter | timeline cache |
| Instagram | feed cache, counter |
| Pinterest | feed cache |
| GitHub | session, job queue |
| Stack Overflow | cache |
| Snapchat | session |
| Stripe | rate limit, cache |

---

## 11. 관련

- [[../mongodb/use-cases]] — Document
- [[../postgresql/use-cases]] — 영속 RDB
- [[redis]] — Redis hub
- [[../database]] — database hub
