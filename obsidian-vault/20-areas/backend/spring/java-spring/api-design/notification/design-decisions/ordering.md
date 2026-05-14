---
title: "Ordering — 알림 순서 보장 필요한가"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:42:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - ordering
---

# Ordering — 알림 순서 보장 필요한가

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — type 별 정책

| Type | 순서 보장 | 이유 |
| --- | --- | --- |
| **CHAT_MESSAGE** | **필요** (room 내) | "안녕 → 오늘 만나자" 순서 뒤바뀌면 의미 X |
| PAYMENT (단일 거래) | 필요 | APPROVED → CANCELED 순 |
| COMMENT_RECEIVED | 약함 | 1 시간 안 다른 댓글 알림 순서 무관 |
| LIKE_RECEIVED | 약함 | 집계 |
| MARKETING | 불필요 | |

---

## 2. 왜 / 안 하면

### 2.1 왜 chat / payment 순서 보장

- chat: 사용자 대화 의미.
- payment: APPROVED 다음 CANCELED — 반대 순 시 사용자 혼란.

### 2.2 안 하면

- chat: "오늘 만나자" 가 먼저 도착 → "안녕" 다음 → 의미 X.
- payment: CANCELED → APPROVED 순 → 환불 받았다 생각하다 다시 결제 받음.

---

## 3. 보장 방법

### 3.1 outbox + single worker

```
같은 chat room 의 모든 메시지 → 같은 worker (partition key=roomId)
→ 순차 처리 보장
```

### 3.2 Kafka partition key

```
Topic: notification.chat
Partition key: roomId
→ 같은 room 의 알림 = 같은 partition = 순서 보장
```

### 3.3 sequence number

```
notification_outbox 에 sequence_number 추가
같은 (user_id, target_type) 안 increment
worker 가 sequence 순 처리
```

---

## 4. 함정

### 함정 1 — concurrent worker 동시 처리
같은 사용자 의 알림 2개 동시 → 순서 X.
→ partition key (user_id) 로 single thread.

### 함정 2 — retry 시 순서 깨짐
A 가 fail retry 후 B 보다 늦게.
→ 같은 partition 만 retry, B 도 대기.

### 함정 3 — channel 별 다른 latency
push 가 email 보다 빠름 → 같은 alert 의 push 가 먼저.
→ 의미적으로 OK (같은 alert 의 다중 채널).

---

## 5. 관련

- [[design-decisions|↑ hub]]
- [[../implementation/outbox-worker-impl]]
- [[kafka-event-driven]]
