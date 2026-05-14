---
title: "Runbook — 5 시나리오"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:26:00+09:00
tags: [backend, java-spring, api-design, chat, operations, runbook]
---

# Runbook — 5 시나리오

**[[operations|↑ hub]]**

---

## 1. WebSocket connection 폭주 (10만+)

```
탐지: chat_ws_active_connections > 80% capacity
조치:
1. node 추가 (k8s scale-out)
2. sticky session 재 분배 (점진)
3. RabbitMQ broker scale-up
4. Redis Pub/Sub 채널 부하 확인
```

## 2. Redis Pub/Sub 장애

```
탐지: Redis ping fail / publish 실패 spike
조치:
1. Redis cluster failover 확인
2. 메시지 전달 불완전 — 사용자 reload 시 REST 로 catchup
3. fallback: 단일 노드 mode (RabbitMQ direct)
```

## 3. RabbitMQ broker 장애 (F5+)

```
탐지: STOMP relay disconnect
조치:
1. RabbitMQ cluster failover
2. 임시: SimpleBroker fallback (in-memory, 단일 노드)
3. 사용자 모두 reconnect 유도
```

## 4. 메시지 send latency spike

```
탐지: send p95 > 2s
조치:
1. DB partition prune 확인
2. Redis INCR latency
3. Broadcast queue size 확인
4. RabbitMQ relay throughput
```

## 5. 신고 / 욕설 spike

```
탐지: chat_report_total spike / 자동 모더 폭주
조치:
1. AI 욕설 필터 (옵션) 적용
2. admin 검토 큐 점검
3. 신고 임계값 조정
```

---

## 관련

- [[operations|↑ hub]]
- [[observability]]
- [[scaling]]
