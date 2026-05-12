---
title: "backend-engineer — 학습 reference 카탈로그"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - reference
  - links
  - backend
  - java
  - spring
  - jpa
---

# backend-engineer — 학습 Reference

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

> backend-engineer 의 책임 (도메인 모델 / 서비스 / API 계약 / 데이터
> 계층) 에 맞춘 외부 reference. Java / Spring 생태계 중심.

**[[links|↑ links 카탈로그]]**

## Java 언어 / 표준

- [OpenJDK 공식 문서](https://openjdk.org/) — JDK 릴리즈 노트 / JEP 인덱스. 신 버전 도입 결정 시 1차 source.
- [Effective Java (Joshua Bloch) — 노트 모음](https://github.com/HugoMatilla/Effective-JAVA-Summary) — 90 항목 요약. 30-resources/books 미러 권장.
- [Modern Java Recipes (Ken Kousen) — GitHub examples](https://github.com/kousen/java_8_recipes) — Stream / Optional / 함수형 패턴.

## Spring 생태계

- [Spring 공식 docs](https://docs.spring.io/spring-framework/reference/) — Spring Framework / Boot / Security / Data 의 권위 문서.
- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/) — auto-configuration / starter / properties 매핑.
- [Spring Academy (Pivotal)](https://spring.academy/) — 공식 무료 학습 코스.
- [Baeldung — Spring tutorials](https://www.baeldung.com/spring-tutorial) — 가장 큰 Spring 실무 패턴 모음 (실제 코드 예 풍부).

## JPA / Database

- [Hibernate 공식 docs](https://hibernate.org/orm/documentation/) — JPA provider 의 표준. N+1 / Fetch 전략 결정 시.
- [Vlad Mihalcea blog](https://vladmihalcea.com/) — JPA / Hibernate 성능의 1위 reference. 트랜잭션 / 락 패턴.
- [JPA Buddy — best practices](https://www.jpa-buddy.com/blog/) — JPA 안티패턴 정리.
- [Database Internals (Alex Petrov) 노트](https://github.com/aaronwangy/Database-Internals) — DB 엔진 동작 원리.

## API 설계

- [RESTful API 디자인 — Microsoft Azure 가이드](https://learn.microsoft.com/azure/architecture/best-practices/api-design) — 실무 REST 설계 표준.
- [Google AIP (API Improvement Proposals)](https://google.aip.dev/) — Google 의 API 설계 원칙 (resource 명명 / pagination / errors).
- [JSON:API 스펙](https://jsonapi.org/) — API 응답 양식 표준.

## 보안 / 인증

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — Top 10 / SQL injection / XSS / JWT 등. 보안 리뷰 시 필수.
- [Spring Security Architecture](https://spring.io/guides/topics/security/) — filter chain / authentication / authorization.

## 추천 운영 / 도구

- `lsp-preflight` 플러그인으로 mypy / checkstyle / spotbugs 사전 차단
- `paste-guard` 가 outbound secret 마스킹 자동
- 트랜잭션 / 락 결정은 `decisions/` 로, 패턴은 `40-patterns/api-patterns/` 로
