---
title: "Redis"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - database
  - redis
---

# Redis

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[database|↑ Database]]**

> 가장 많이 쓰이는 in-memory store. 캐시 / 큐 / pub-sub / 분산 락.

## Reference 링크

- [Redis 공식 docs](https://redis.io/docs/) — 공식 권위 docs
- [Redis Commands](https://redis.io/commands/) — 모든 명령어 reference
- [Redis Patterns](https://redis.io/docs/manual/patterns/) — pub-sub / lock / counter 패턴
- [Redis Persistence (RDB / AOF)](https://redis.io/docs/manual/persistence/) — 영속화 전략
- [Redis Sentinel / Cluster](https://redis.io/docs/manual/scaling/) — 고가용성 / 샤딩
- [Redis Streams](https://redis.io/docs/data-types/streams/) — 메시징 큐 (Kafka 대안)
- [Redis University](https://university.redis.com/) — 공식 무료 강의
- [Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/current/reference/html/) — Spring 통합
- [Redisson docs](https://github.com/redisson/redisson/wiki/Table-of-Content) — Java Redis 클라이언트 (분산 락 / 컬렉션)
- [Lettuce docs](https://lettuce.io/core/release/reference/index.html) — Spring Boot 기본 Java 클라이언트
- [Redis Memory Optimization](https://redis.io/docs/management/optimization/memory-optimization/) — 메모리 최적화
- [Redis 캐시 패턴 (cache-aside)](https://redis.io/learn/howtos/solutions/caching-architecture/common-caching-patterns) — cache-aside / read-through
- [Distributed lock (Redlock)](https://redis.io/docs/manual/patterns/distributed-locks/) — 분산 락 알고리즘
- [Redis Best Practices](https://redis.io/docs/manual/patterns/) — 운영 모범 사례
- [우아한형제들 - Redis 활용](https://techblog.woowahan.com/?s=redis) — 한국 회사 사례
