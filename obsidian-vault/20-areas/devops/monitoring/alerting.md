---
title: "Alerting — AlertManager / PagerDuty / Opsgenie ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:45:00+09:00
tags: [devops, monitoring, alerting, alertmanager, pagerduty]
---

# Alerting — AlertManager / PagerDuty / Opsgenie ★

**[[monitoring|↑ monitoring]]**

---

## 1. 왜 필요

- monitoring 만으로는 부족 — **사람이 깨워야** 대응.
- 정확한 alert = SRE 의 핵심 (false positive = alert fatigue).
- on-call rotation + escalation policy 필수.

---

## 2. 도구

| | 역할 | 무엇 |
| --- | --- | --- |
| **AlertManager** | alert routing | Prom 기반, OSS |
| **Grafana Alerting** | alert + viz 통합 | UI 친화 |
| **PagerDuty** | on-call / escalation | 통화 / SMS / app push |
| **Opsgenie** | on-call (Atlassian) | PagerDuty 경쟁 |
| **VictorOps / Splunk On-Call** | | |
| **Slack / Teams** | low-severity 알림 | escalation 안 됨 |

---

## 3. 흐름

```
[Prometheus] → 룰 평가 → 발생 → [AlertManager]
                                   │
                                   ├─ silence / dedup / group
                                   │
                                   ▼
                                [PagerDuty] / Slack / Email
                                   │
                                   ▼
                                on-call 엔지니어 (SMS / call)
```

---

## 4. Prometheus alert rule

```yaml
# PrometheusRule (CRD)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata: {name: web-alerts}
spec:
  groups:
    - name: web
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_server_requests_seconds_count{status=~"5..", app="web"}[5m]))
            / sum(rate(http_server_requests_seconds_count{app="web"}[5m]))
            > 0.05
          for: 5m
          labels:
            severity: page
          annotations:
            summary: "Web 5xx error > 5%"
            description: "Error rate is {{ $value | humanizePercentage }}"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[10m]) > 0.5
          for: 10m
          labels: {severity: page}
```

---

## 5. AlertManager 설정

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'slack-default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s          # 같은 그룹 첫 alert wait
  group_interval: 5m       # 추가 alert 묶기 간격
  repeat_interval: 4h      # 동일 alert 반복 알림 간격
  routes:
    - match: {severity: page}
      receiver: 'pagerduty'
      continue: true
    - match: {severity: warning}
      receiver: 'slack-warning'

receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: <key>
  - name: 'slack-default'
    slack_configs:
      - api_url: <webhook>
        channel: '#alerts'

inhibit_rules:
  - source_match: {severity: critical}
    target_match: {severity: warning}
    equal: [alertname, cluster]
```

---

## 6. severity 표준

| severity | 대응 | 예 |
| --- | --- | --- |
| **page (P0/P1)** | 즉시 콜 (밤 2시도) | 서비스 down / DB 마스터 fail |
| **page (P2)** | 1시간 내 | error rate spike |
| **warning** | 다음날 | disk 70%, slow query |
| **info** | log only | deploy / scale event |

→ **page 는 정말 깨워야 할 때만**.

---

## 7. alert 디자인 — SLO-based

```yaml
# burn rate alert (Google SRE)
- alert: ErrorBudgetBurn
  expr: |
    (
      sum(rate(http_errors_total[1h])) / sum(rate(http_total[1h]))
    ) > (14.4 * 0.001)   # 14.4x burn, 99.9% SLO
  for: 5m
  labels: {severity: page}
```

→ SLO 기반 alert = noisy 한 metric-based 보다 훨씬 정확.

---

## 8. PagerDuty 구성

```
1. Service 생성 (web-api, db, ...)
2. Escalation Policy
   - Level 1: primary on-call (15min response)
   - Level 2: secondary on-call
   - Level 3: manager
3. Schedule
   - Weekly rotation
   - Layer: APAC + EU + US (follow-the-sun)
4. Integration: AlertManager / Datadog / Grafana
```

---

## 9. 함정 (alert fatigue)

1. **너무 많은 alert** → 무시 (boy-who-cried-wolf).
2. **fixable 하지 않은 alert** — "log 가 많다" — 사람이 할 수 있는 일 없음.
3. **for: 0s** — flapping → 즉시 resolve → noise.
4. **runbook 없음** — 새벽 3시에 무엇을 해야 할지 모름.
5. **dedup / group 없음** — 1개 장애로 100개 page.
6. **escalation 없음** — 1차 응답자 자고 있으면 무한 대기.
7. **silence 없음** — 정기 점검 중에도 page.

---

## 10. 관련

- [[monitoring|↑ monitoring]]
- [[prometheus]]
- [[slos-and-sli]]
- [[../sre/sre|↗ sre]]
