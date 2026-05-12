---
title: "Hibernate / JPA 패턴"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - backend
  - hibernate-jpa
---

# Hibernate / JPA 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> Hibernate 의 내부 동작 + JPA 표준 패턴 (N+1 / fetch / 락 / 캐시 / 2차 캐시).

## Reference 링크

- [Hibernate 공식 docs](https://hibernate.org/orm/documentation/) — 권위 docs
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html) — Hibernate ORM 완전 가이드
- [Vlad Mihalcea blog](https://vladmihalcea.com/) — Hibernate 성능 1위 reference
- [Vlad — N+1 problem](https://vladmihalcea.com/n-plus-1-query-problem/) — N+1 의 정의 + 해결
- [Vlad — Fetch types](https://vladmihalcea.com/eager-fetching-is-a-code-smell/) — EAGER 의 함정
- [JPA Buddy — Best practices](https://www.jpa-buddy.com/blog/) — JPA 모범 사례
- [Thoughts on Java](https://thorben-janssen.com/) — Hibernate 깊은 글
- [JPQL spec](https://jakarta.ee/specifications/persistence/3.1/) — JPA 3.1 표준
- [Criteria API 가이드](https://www.baeldung.com/hibernate-criteria-queries) — Criteria 동적 쿼리
- [Hibernate 2차 캐시](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#caching) — 캐시 전략
- [Pessimistic vs Optimistic Lock](https://www.baeldung.com/jpa-pessimistic-locking) — 락 전략
- [Hibernate Envers](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#envers) — 엔티티 감사 (versioning)
- [High-Performance Java Persistence (Vlad 책)](https://vladmihalcea.com/books/high-performance-java-persistence/) — JPA / Hibernate 표준 책
- [Java Persistence with Spring Data and Hibernate (Manning)](https://www.manning.com/books/java-persistence-with-spring-data-and-hibernate) — JPA 표준 책
- [Spring + JPA pitfalls (Reflectoring)](https://reflectoring.io/jpa-pitfalls/) — 함정 모음
