---
title: "Caching 전략"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - backend
  - caching
---

# Caching 전략

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> 어플리케이션 캐시 / 분산 캐시 / cache-aside / read-through / write-back 패턴.

## Reference 링크

- [Caching Patterns (AWS)](https://aws.amazon.com/caching/) — 캐시 패턴 정리
- [Cache-aside vs Read-through (Hazelcast)](https://docs.hazelcast.com/hazelcast/5.3/data-structures/caching-patterns) — 캐시 패턴 비교
- [Spring Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html) — Spring @Cacheable
- [Caffeine Cache](https://github.com/ben-manes/caffeine) — Java 의 표준 in-memory 캐시
- [Redis 캐시 패턴](https://redis.io/learn/howtos/solutions/caching-architecture/common-caching-patterns) — Redis 공식 캐시 가이드
- [CDN Cache Best Practices (Cloudflare)](https://developers.cloudflare.com/cache/concepts/) — CDN 캐시 동작
- [HTTP Cache (RFC 9111)](https://www.rfc-editor.org/rfc/rfc9111.html) — HTTP 캐시 표준
- [Cache Stampede](https://en.wikipedia.org/wiki/Cache_stampede) — thundering herd 문제
- [Cache eviction policies](https://en.wikipedia.org/wiki/Cache_replacement_policies) — LRU / LFU / ARC
- [Awesome caching](https://github.com/MoonshotCollective/awesome-caching) — 캐시 자료 인덱스
- [Hibernate 2차 캐시](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#caching) — ORM 캐시
- [Varnish cache](https://varnish-cache.org/) — reverse proxy 캐시
- [Cache invalidation (Phil Karlton)](https://martinfowler.com/bliki/TwoHardThings.html) — 캐시 무효화의 어려움
- [Netflix EVCache](https://github.com/Netflix/EVCache) — Netflix 의 분산 캐시
- [Memcached docs](https://memcached.org/) — 단순 분산 캐시
