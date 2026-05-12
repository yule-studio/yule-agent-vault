---
title: "Spring Cloud Gateway"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T22:30:00+09:00
tags:
  - reference
  - links
  - backend
  - spring-cloud-gateway
---

# Spring Cloud Gateway

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> API 게이트웨이 / 라우팅 / 필터 / Rate limit.

## Reference 링크

- [Spring Cloud Gateway Reference](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/) — 권위 docs
- [Gateway predicates](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gateway-request-predicates-factories) — route 조건
- [Gateway filters](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gatewayfilter-factories) — request/response 필터
- [Rate Limiting](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#the-requestratelimiter-gatewayfilter-factory) — Redis 기반 rate limit
- [Spring Cloud Gateway Source](https://github.com/spring-cloud/spring-cloud-gateway) — 소스 + 이슈
- [Baeldung — Cloud Gateway](https://www.baeldung.com/spring-cloud-gateway) — 실무 패턴
- [Reactive Gateway 가이드](https://www.baeldung.com/spring-cloud-gateway-routing-predicate-factories) — predicate 가이드
- [Custom Global Filter](https://www.baeldung.com/spring-cloud-custom-gateway-filters) — 커스텀 필터
- [Gateway vs Zuul](https://spring.io/blog/2018/06/25/spring-cloud-gateway-vs-zuul) — 레거시 Zuul 대비
- [Spring Cloud Gateway Server MVC](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/server-mvc.html) — MVC 버전 (non-reactive)
- [Circuit breaker 통합](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#spring-cloud-circuitbreaker-filter-factory) — 회로 차단
- [Kong vs Spring Cloud Gateway](https://konghq.com/blog/api-gateway-vs-service-mesh) — 비교 글
- [Spring Cloud Gateway + JWT](https://www.baeldung.com/spring-cloud-gateway-oauth2) — OAuth2 통합
- [Spring Cloud Gateway 예시 (samples)](https://github.com/spring-cloud-samples/spring-cloud-gateway-sample) — 공식 sample
- [Awesome API Gateway](https://github.com/spcanelon/awesome-api-gateway) — gateway 자료