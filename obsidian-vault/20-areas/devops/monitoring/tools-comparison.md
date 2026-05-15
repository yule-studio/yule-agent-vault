---
title: "Monitoring 도구 비교 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:30:00+09:00
tags: [devops, monitoring, comparison]
---

# Monitoring 도구 비교 ★

**[[monitoring|↑ monitoring]]**

---

## 1. 매트릭스

| | **Prometheus+Grafana** ★ | **Datadog** | **Grafana Cloud** | New Relic | Dynatrace | ELK | CloudWatch |
| --- | --- | --- | --- | --- | --- | --- | --- |
| OSS | O | X | (Grafana OSS) | X | X | O | X |
| metric | ✓ | ✓ | ✓ | ✓ | ✓ | (Metricbeat) | ✓ |
| log | (Loki) | ✓ | ✓ (Loki) | ✓ | ✓ | ✓ | ✓ |
| trace | (Tempo) | ✓ | ✓ (Tempo) | ✓ | ✓ | ✓ APM | X-Ray |
| 비용 | self-host (인프라) | $15-23 / host | $$ (host / metric) | $$ | $$$$ | self / Elastic Cloud | metric/log 사용량 |
| 학습 | PromQL | 쉬움 | 같음 | 쉬움 | 쉬움 | 어려움 | 쉬움 |
| 본 vault | ★ | enterprise | option | enterprise | enterprise large | log heavy | AWS only |

---

## 2. 선택 가이드

| 상황 | 추천 |
| --- | --- |
| **시작 / OSS / 비용 ↓** | Prometheus + Grafana + Loki (self-host) |
| **운영 부담 ↓** | Grafana Cloud (관리형) |
| **enterprise + 통합 SaaS** | Datadog |
| **AWS only** | CloudWatch (시작) → Grafana for visual |
| **legacy 로그 중심** | ELK |
| **APM 강화** | Datadog APM / New Relic |
| **k8s native** | Prometheus + kube-state-metrics |

---

## 3. Prometheus 의 강점

- ✓ Pull 모델 (target 조회) — service discovery 자연스러움.
- ✓ PromQL — 강력한 query (rate / histogram).
- ✓ k8s 표준 (kube-prometheus-stack).
- ✗ long-term storage 약 → Thanos / Cortex / Mimir.
- ✗ log / trace 별도 (Loki / Tempo 와 조합).

---

## 4. Datadog 의 강점

- ✓ 통합 UI (metric + log + trace + RUM).
- ✓ agent 설치 = 자동 collection.
- ✓ 운영 부담 0.
- ✗ 가격 ($$ per host / log volume).
- ✗ lock-in.

---

## 5. 본 vault 의 결정

```
F0~F3: Prometheus + Grafana + Loki (self-host k8s)
F4+: + Tempo (trace) + AlertManager
F8+: Grafana Cloud 검토 (운영 부담 ↓)
또는: Datadog (enterprise, 매출 / 신뢰 critical)
```

---

## 6. 관련

- [[monitoring|↑ monitoring]]
- [[prometheus]]
- [[grafana]]
- [[loki]]
