---
title: "backend — Reference 카탈로그"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - index
  - backend
---

# backend — Reference 카탈로그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 카탈로그 |

**[[engineering|↑ Engineering]]**

> backend-engineer 의 책임 (도메인 모델 / 서비스 / API 계약 / 데이터 계층 / 보안) 영역별 세분 카탈로그.

## 카탈로그

| 주제 | 진입점 | 한 줄 |
| --- | --- | --- |
| Java 언어 / JVM / 표준 라이브러리 | [[java]] | Java 8/11/17/21 / JVM 튜닝 / GC |
| Spring Framework 코어 | [[spring-framework]] | IoC / AOP / Bean lifecycle |
| Spring Boot | [[spring-boot]] | auto-configuration / starter / actuator |
| Spring Security | [[spring-security]] | 인증 / 인가 / OAuth2 / JWT |
| Spring Data JPA | [[spring-data-jpa]] | Repository / Query / projection |
| Hibernate / JPA 패턴 | [[hibernate-jpa]] | N+1 / fetch / 락 / 캐시 |
| MyBatis | [[mybatis]] | XML mapper / dynamic SQL |
| API 설계 (REST / GraphQL / gRPC) | [[api-design]] | 리소스 명명 / pagination / errors |
| Database (PostgreSQL / MySQL / Redis / MongoDB) | [[database]] | RDB / NoSQL / 캐시 |
| Messaging (Kafka / RabbitMQ / Redis Streams) | [[messaging]] | 비동기 / 이벤트 / outbox |
| Transaction / Concurrency 패턴 | [[transaction-patterns]] | ACID / 격리 수준 / Saga |
| Caching 전략 | [[caching]] | Redis / Caffeine / read-through |
| 보안 모범 사례 | [[security-best-practices]] | OWASP Top 10 / SQL Injection / XSS |
| 백엔드 관측 | [[observability-backend]] | Micrometer / OpenTelemetry / 로깅 |
| 성능 튜닝 | [[performance-tuning]] | profiling / JVM heap / DB index |