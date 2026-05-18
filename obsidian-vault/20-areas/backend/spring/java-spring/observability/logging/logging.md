---
title: "logging — Spring 로그 관리 Sub-Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T08:30:00+09:00
tags: [backend, java-spring, observability, logging, hub]
home_hub: observability
related:
  - "[[../observability]]"
  - "[[fundamentals]]"
  - "[[mdc-logging-filter]]"
  - "[[logback-configuration]]"
  - "[[elk-stack]]"
  - "[[../monitoring/monitoring]]"
---

# logging — Spring 로그 관리 Sub-Hub

**[[../observability|↑ observability]]**

> Spring Boot 의 로그 관리 — 왜 / 무엇 / Logback / MDC / ELK — 4 단계 완성.

---

## 1. 영역 정의

본 영역은 Spring Boot 응용의 로그 관리를 첫 console 출력부터 ELK 중앙 수집까지 다룬다.

본 영역이 정의하는 것:
- 로그의 의미 / Level / 8 구성요소
- request 추적 (MDC + Servlet Filter)
- Logback 설정 (Appender / Rolling / 압축)
- ELK Stack 통합

본 영역이 정의하지 않는 것:
- 메트릭 (Actuator / Prometheus / Grafana) — [[../monitoring/monitoring]]
- DevOps 측 ELK 운영 깊이 — [[../../../../../devops/monitoring/monitoring]]

---

## 2. 노트 인덱스 — 순서대로

| 단계 | 노트 | 다루는 주제 |
| --- | --- | --- |
| 1 | [[fundamentals]] | 왜 로그 필요 / 무엇을 기록 / Level (TRACE-FATAL) / 8 구성요소 / SLF4J 사용법 |
| 2 | [[mdc-logging-filter]] | MDC + UUID + Servlet Filter / 외부 trace id 전파 / `@Async` 의 MDC 보존 |
| 3 | [[logback-configuration]] | Logback Appender / RollingFile / `.log.gz` 압축 / 환경별 분리 / JSON 출력 |
| 4 | [[elk-stack]] | Elasticsearch + Logstash + Kibana — Docker Compose / pipeline / index 설계 / Kibana 대시보드 |

---

## 3. 학습 순서

| 단계 | 작업 | 검증 |
| --- | --- | --- |
| 1 | [[fundamentals]] — SLF4J `log.info("...", arg)` 패턴 | 로그가 Level 별로 console 에 보임 |
| 2 | [[mdc-logging-filter]] — `MdcLoggingFilter` 추가 | `grep request_id=...` 으로 한 request 의 모든 로그 추출 |
| 3 | [[logback-configuration]] — `logback-spring.xml` + RollingFile | 일별 회전 + .gz 압축 자동 |
| 4 | [[elk-stack]] — docker-compose + Logstash TCP | Kibana Discover 에서 로그 검색 가능 |

각 단계가 이전 단계 가정. 1-3 은 Spring 측, 4 는 인프라 추가.

---

## 4. 관련

- [[../observability|↑ observability]]
- [[fundamentals]]
- [[mdc-logging-filter]]
- [[logback-configuration]]
- [[elk-stack]]
- [[../monitoring/monitoring|→ monitoring sub-hub]]
- [[../../../../../devops/monitoring/monitoring|↗ DevOps monitoring]]
