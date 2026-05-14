---
title: "Ordering 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:34:00+09:00
tags: [backend, java-spring, api-design, chat, pitfalls, ordering]
---

# Ordering 함정

**[[pitfalls|↑ hub]]**

---

1. **client timestamp 사용** → clock skew 시 순서 X.
2. **(room_id, seq) UNIQUE 없음** → race 시 중복.
3. **Redis INCR fail 시 fallback X** → 메시지 INSERT 영구 실패.
4. **out-of-order broadcast** (Pub/Sub partition) → FE 가 seq 정렬.
5. **FE buffer 무한 대기** (gap message) → 5s timeout + REST refetch.
6. **partition prune 안 됨** → 모든 partition scan.
7. **단일 시퀀스 (room 무관)** → 한 사용자가 다 hot.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/message-ordering]]
