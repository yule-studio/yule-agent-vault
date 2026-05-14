---
title: "Observability — metric + alert"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:24:00+09:00
tags: [backend, java-spring, api-design, chat, operations, observability]
---

# Observability — metric + alert

**[[operations|↑ hub]]**

---

## 1. Metric

| Metric | Type | Tag |
| --- | --- | --- |
| `chat_ws_active_connections` | gauge | node |
| `chat_ws_connect_total` | counter | result (success/fail) |
| `chat_ws_disconnect_total` | counter | reason |
| `chat_message_send_total` | counter | room_type, message_type |
| `chat_message_send_duration_seconds` | histogram | room_type |
| `chat_message_broadcast_duration_seconds` | histogram | — |
| `chat_redis_pubsub_msg_total` | counter | channel |
| `chat_room_member_count` | gauge | room_id (top N only) |
| `chat_presence_online_count` | gauge | — |
| `chat_message_offline_push_total` | counter | — |
| `chat_attachment_upload_total` | counter | type |
| `chat_rate_limit_rejected_total` | counter | — |

---

## 2. Alert

| 조건 | 채널 |
| --- | --- |
| WS connection > 80% capacity | #infra (scale-out 검토) |
| send latency p95 > 2s | #infra |
| Redis Pub/Sub fail | #infra (P0) |
| RabbitMQ broker fail | #infra (P0) |
| 신고 spike | #moderation |
| rate limit rejected > 100/min | #abuse |

---

## 3. Grafana dashboard

- Connections by node (timeseries)
- Message throughput
- Latency p50/p95/p99
- Redis Pub/Sub lag
- Active rooms / users

---

## 관련

- [[operations|↑ hub]]
- [[scaling]]
- [[../testing/load-tests]]
