---
title: "Scaling — sticky + Redis cluster + RabbitMQ ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:28:00+09:00
tags: [backend, java-spring, api-design, chat, operations, scaling]
---

# Scaling — sticky + Redis cluster + RabbitMQ ★

**[[operations|↑ hub]]**

> 설계: [[../design-decisions/scale-strategy]].

---

## 1. 단계

| Capacity | 인프라 |
| --- | --- |
| ~ 1만 동시 | 단일 노드 + SimpleBroker |
| ~ 5만 | 5 노드 + sticky + Redis Pub/Sub |
| ~ 50만 | 20+ 노드 + RabbitMQ STOMP relay cluster |
| 100만+ | + Kafka + region 분리 |

---

## 2. AWS 배포 예시

```yaml
# AWS ALB
target-group:
  protocol: HTTP
  stickiness:
    type: app_cookie
    cookie-name: CHAT_SESSION
    duration: 1d
  health-check:
    path: /actuator/health
    interval: 10s

# EC2 / EKS
chat-app (k8s deployment):
  replicas: 5
  resources:
    limits:
      memory: 4Gi
      cpu: 2

# ElastiCache Redis (cluster mode)
shards: 3
replicas-per-shard: 2

# Amazon MQ (RabbitMQ) — F5+
broker:
  instance-type: mq.m5.large
  multi-az: true
```

---

## 3. node 별 한도

| 항목 | 값 |
| --- | --- |
| OS file descriptor | `ulimit -n 65536` |
| JVM heap | 4GB (1만 connection) |
| Tomcat threads | 200 (sync) 또는 WebFlux Netty (async, 권장) |
| WebSocket buffer size | `server.tomcat.max-http-form-post-size=1MB` |

---

## 4. graceful disconnect (rolling deploy)

```
1. ALB 에서 노드 drain (new connection X)
2. 30s 후 STOMP DISCONNECT broadcast (모든 active session)
3. 사용자 FE 가 reconnect (다른 노드)
4. 노드 종료
```

→ 사용자 끊김 < 5s.

---

## 관련

- [[operations|↑ hub]]
- [[../design-decisions/scale-strategy]]
- [[runbook]]
