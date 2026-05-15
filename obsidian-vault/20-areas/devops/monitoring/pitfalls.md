---
title: "monitoring — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:53:00+09:00
tags: [devops, monitoring, pitfalls]
---

# monitoring — 함정 모음

**[[monitoring|↑ monitoring]]**

---

## 1. metric

1. **high cardinality label** (user_id, full path) → series 폭주 → Prom OOM.
2. **counter 가 아닌 gauge** — `_total` 안 붙임 → rate 계산 불가.
3. **percentile 평균** — `avg(p95)` 는 수학적으로 잘못 → `histogram_quantile`.
4. **histogram bucket 부적절** — percentile 부정확.
5. **success-only metric** — error rate 계산 불가.

---

## 2. log

1. **모든 로그 INFO** — log 폭주, 정작 ERROR 묻힘.
2. **PII 로깅** — 이메일 / 카드번호 평문 (GDPR / PCI 위반).
3. **structured 아님** — text 만 → 검색 / 필터 불가.
4. **stack trace 없음** — exception 만 message, 분석 불가.
5. **MDC 없음** — request 간 log 구분 불가 (trace_id, user_id).

---

## 3. trace

1. **sampling 100%** prod — 비용 폭주.
2. **trace_id 전파 누락** — Kafka producer/consumer 사이 header 안 넘김.
3. **너무 많은 span** — 모든 method 에 span → noise.
4. **log ↔ trace 안 연결** — exemplar / MDC 누락.
5. **PII span attribute** — full URL / body 그대로.

---

## 4. alerting

1. **alert fatigue** — 너무 많은 page → 무시 → 정작 중요한 거 놓침.
2. **fixable 하지 않은 alert** — "log 많음", "GC 발생" — 사람이 할 게 없음.
3. **runbook 없음** — 새벽 3시에 무엇을 해야 할지 모름.
4. **flapping** — `for: 0s` → resolve / fire 무한 반복.
5. **silence 없음** — 정기 점검 중에도 page.
6. **escalation 없음** — 1차 응답자 자고 있으면 무한 대기.

---

## 5. dashboard

1. **너무 많은 dashboard** — 어디 봐야 할지 모름 → folder / tag 정리.
2. **무거운 query** — wide time range → load.
3. **business / 시스템 분리** — 둘이 따로 → context switching.
4. **dashboard as code 안 함** — UI 에서만 만들면 disaster recovery 불가.
5. **annotation 없음** — deploy / incident 마커.

---

## 6. SLO

1. **SLO 너무 많음** — 우선순위 모호 → 서비스당 3-5개.
2. **availability 100% 목표** — 비용 무한 + 기능 stagnation.
3. **사용자 관점 아님** — backend metric 만 — frontend (CDN) 까지 안 봄.
4. **error budget freeze 정책 없음** — SLO 의 의미 없음.
5. **window 너무 짧음** (1일) — flapping.

---

## 7. observability 일반

1. **3 pillars 분리** — Prometheus, Loki, Tempo 따로 — exemplar / trace_id 로 연결 안 함.
2. **dev / prod metric 분리 안 됨** — `env=` label 누락.
3. **vendor lock-in** — Datadog 만 의존 → 비용 폭주 시 마이그레이션 지옥.
4. **on-call 만 알아봄** — 다른 팀이 dashboard 볼 줄 모름.
5. **postmortem 없음** — incident 반복.

---

## 8. 관련

- [[monitoring|↑ monitoring]]
- [[../sre/sre|↗ sre]]
