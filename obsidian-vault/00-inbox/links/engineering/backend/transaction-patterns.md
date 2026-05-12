---
title: "Transaction / Concurrency 패턴"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - backend
  - transaction-patterns
---

# Transaction / Concurrency 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> ACID / 격리 수준 / 락 / 분산 트랜잭션 (Saga / 2PC) / 멱등성.

## Reference 링크

- [ACID 정의 (Wikipedia)](https://en.wikipedia.org/wiki/ACID) — ACID 의 표준 정의
- [Transaction Isolation Levels (PG docs)](https://www.postgresql.org/docs/current/transaction-iso.html) — 격리 수준의 정석
- [Spring Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html) — @Transactional propagation
- [Optimistic vs Pessimistic Locking](https://www.baeldung.com/jpa-pessimistic-locking) — 락 전략 비교
- [Saga Pattern (microservices.io)](https://microservices.io/patterns/data/saga.html) — 분산 트랜잭션 패턴
- [Two-Phase Commit (DDIA 9장)](https://dataintensive.net/) — 2PC 의 정의
- [Idempotency keys (Stripe)](https://stripe.com/blog/idempotency) — 멱등성 패턴 사례
- [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) — 메시징 + 트랜잭션
- [Vlad — Concurrency Patterns](https://vladmihalcea.com/category/concurrency/) — JPA 락 카테고리
- [Distributed Locks (Redlock)](https://redis.io/docs/manual/patterns/distributed-locks/) — Redis 분산 락
- [Concurrency Patterns (Bert Bates)](https://www.oreilly.com/library/view/concurrency-patterns/) — Java 동시성 책
- [Java Concurrency in Practice (책)](https://jcip.net/) — Java 동시성 표준 책
- [Read uncommitted / Phantom reads](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED) — 이상 현상 정의
- [Snapshot Isolation](https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/) — 논문 (Berenson et al)
- [Eventual Consistency (Amazon)](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html) — Werner Vogels 의 글
