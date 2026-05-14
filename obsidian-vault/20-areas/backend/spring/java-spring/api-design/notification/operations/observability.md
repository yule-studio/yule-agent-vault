---
title: "Observability — Prometheus + Grafana"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:18:00+09:00
tags: [backend, java-spring, api-design, notification, operations, observability]
---

# Observability — Prometheus + Grafana

**[[operations|↑ hub]]**

---

## 1. 메트릭

| Metric | Type | Tag |
| --- | --- | --- |
| `notification_outbox_pending_count` | gauge | — |
| `notification_outbox_inserted_total` | counter | type |
| `notification_sent_total` | counter | type, channel, status |
| `notification_send_duration_seconds` | histogram | channel |
| `notification_dlq_total` | counter | type, reason |
| `notification_dlq_count` | gauge | — |
| `notification_retry_attempts_total` | counter | channel, attempt |
| `fcm_response_total` | counter | status (success/unregistered/quota/...) |
| `apns_response_total` | counter | status |
| `ses_bounce_total` | counter | type (hard/soft/complaint) |
| `device_active_count` | gauge | platform, channel |
| `device_invalidated_total` | counter | reason |

---

## 2. 알람

| 조건 | 채널 | 우선 |
| --- | --- | --- |
| outbox pending > 1000 | #infra | P1 |
| outbox pending > 10000 | #infra | P0 |
| DLQ count > 10 / 10min | #infra | P1 |
| FCM `THIRD_PARTY_AUTH_ERROR` | #infra | P0 (service account 문제) |
| SES bounce rate > 5% | #email | P0 (계정 정지 위험) |
| send duration p95 > 5s | #infra | P1 |
| device invalidated spike > 100 / hour | #infra | P2 |

---

## 3. Grafana 대시보드

- 발송량 (per channel, per type) timeseries.
- DLQ 누적 + replay rate.
- channel 별 latency p50/p95/p99.
- device active count by platform.
- bounce / complaint trend.

---

## 4. 관련

- [[operations|↑ hub]]
- [[runbook]]
- [[dlq-replay]]
