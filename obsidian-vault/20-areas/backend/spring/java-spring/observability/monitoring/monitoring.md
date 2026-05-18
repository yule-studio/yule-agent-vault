---
title: "monitoring — Spring 메트릭 모니터링 Sub-Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T08:35:00+09:00
tags: [backend, java-spring, observability, monitoring, hub]
home_hub: observability
related:
  - "[[../observability]]"
  - "[[actuator-micrometer]]"
  - "[[prometheus-grafana]]"
  - "[[../logging/logging]]"
---

# monitoring — Spring 메트릭 모니터링 Sub-Hub

**[[../observability|↑ observability]]**

> Spring Boot 의 메트릭 모니터링 — Actuator 노출부터 Grafana Dashboard + Discord Alert 까지 2 단계 완성.

---

## 1. 영역 정의

본 영역은 Spring Boot 응용의 메트릭 모니터링을 Actuator 노출부터 Grafana Alert 까지 다룬다.

본 영역이 정의하는 것:
- Spring Actuator endpoint + Micrometer instrument
- Prometheus 의 메트릭 수집
- Grafana 의 대시보드 + Alert
- Discord webhook 통합

본 영역이 정의하지 않는 것:
- 로그 (Logback / ELK) — [[../logging/logging]]
- 분산 trace — 별도 (OpenTelemetry / Tempo)
- DevOps 측 monitoring 운영 깊이 — [[../../../../../devops/monitoring/monitoring]]

---

## 2. 노트 인덱스 — 순서대로

| 단계 | 노트 | 다루는 주제 |
| --- | --- | --- |
| 1 | [[actuator-micrometer]] | Spring Actuator endpoint 16+ / Micrometer 4 instrument (Counter / Gauge / Timer / DistributionSummary) / 자동 측정 (HTTP / JVM / Hikari) / `/actuator/prometheus` |
| 2 | [[prometheus-grafana]] | Prometheus 설치 / scrape 설정 / PromQL 12 패턴 / Grafana 대시보드 4 panel / Alert + Discord webhook |

---

## 3. 학습 순서

| 단계 | 작업 | 검증 |
| --- | --- | --- |
| 1 | [[actuator-micrometer]] — Actuator 의존성 + `/actuator/prometheus` 노출 | `curl /actuator/prometheus` 응답 확인 |
| 2 | [[prometheus-grafana]] — Prometheus + Grafana docker-compose | target health=up + 4 골든 시그널 panel |
| 3 | Discord webhook + Alert rule | 5xx > 5% 시 Discord 알림 도착 |

---

## 4. 관련

- [[../observability|↑ observability]]
- [[actuator-micrometer]]
- [[prometheus-grafana]]
- [[../logging/logging|→ logging sub-hub]]
- [[../../../../../devops/monitoring/monitoring|↗ DevOps monitoring]]
- [[../../../../../devops/sre/sre|↗ DevOps SRE]]
