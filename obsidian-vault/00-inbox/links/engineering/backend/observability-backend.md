---
title: "백엔드 관측 — Micrometer / OTel / 로깅"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - backend
  - observability-backend
---

# 백엔드 관측 — Micrometer / OTel / 로깅

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> 메트릭 / 분산 추적 / 로깅 + 백엔드 관점 (Spring 통합 중심).

## Reference 링크

- [Micrometer 공식 docs](https://micrometer.io/docs) — JVM 메트릭 표준 (Spring Boot 기본)
- [Spring Boot Actuator + Micrometer](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics) — Spring 메트릭 통합
- [OpenTelemetry Java](https://opentelemetry.io/docs/instrumentation/java/) — OTel Java instrumentation
- [Spring Cloud Sleuth (→ Micrometer Tracing)](https://docs.spring.io/spring-cloud-sleuth/docs/3.1.11/reference/html/) — 분산 추적 (3.x deprecated, Micrometer Tracing 으로 이전)
- [SLF4J + Logback](https://logback.qos.ch/manual/) — Java 로깅 표준
- [JSON logging — Logstash encoder](https://github.com/logfellow/logstash-logback-encoder) — 구조화 로그
- [ELK Stack 가이드](https://www.elastic.co/guide/index.html) — Elasticsearch + Logstash + Kibana
- [Grafana Loki](https://grafana.com/docs/loki/latest/) — 로그 집계
- [Jaeger 분산 추적](https://www.jaegertracing.io/docs/) — 분산 추적 표준
- [Zipkin](https://zipkin.io/) — 분산 추적 (Jaeger 대안)
- [Prometheus + Spring Boot](https://prometheus.github.io/client_java/) — Java 클라이언트
- [Awesome Observability](https://github.com/adriannovegil/awesome-observability) — 관측 자료 인덱스
- [Three Pillars of Observability](https://www.honeycomb.io/blog/observability-101-terminology-and-concepts) — 메트릭 / 로그 / 추적
- [Datadog Spring Boot 통합](https://docs.datadoghq.com/integrations/spring_boot/) — 상용 APM 통합
- [New Relic Java APM](https://docs.newrelic.com/docs/apm/agents/java-agent/getting-started/introduction-new-relic-java/) — 상용 APM
