---
title: "monitoring — 실습"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:55:00+09:00
tags: [devops, monitoring, practice]
---

# monitoring — 실습

**[[../monitoring|↑ monitoring]]**

---

## 실습 시리즈

1. [[01-kube-prometheus-stack-setup]] — k8s 에 kube-prometheus-stack 설치 + Spring app scrape
2. [[02-loki-log-aggregation]] — Loki + Promtail + Grafana 통합
3. [[03-otel-distributed-tracing]] — OTel agent + Tempo + log-trace 연결

---

## 진행 순서 권장

- Day 1: 01 (Prometheus + Grafana + Spring metric scrape)
- Day 2: 02 (Loki log 통합)
- Day 3: 03 (OTel + Tempo + trace_id MDC)

---

## 사전 준비물

- minikube 또는 kind cluster (k8s 로컬)
- helm 3.x
- kubectl
- Spring Boot sample app (actuator + micrometer-registry-prometheus 의존성)
- Java 21+
