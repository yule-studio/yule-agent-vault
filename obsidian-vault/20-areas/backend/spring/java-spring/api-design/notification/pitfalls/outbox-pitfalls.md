---
title: "Outbox 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:26:00+09:00
tags: [backend, java-spring, api-design, notification, pitfalls]
---

# Outbox 함정

**[[pitfalls|↑ hub]]**

---

1. **직접 외부 호출** (outbox 없이) → DB commit 후 FCM/APNs timeout 시 event 손실.
2. **트랜잭션 안 외부 호출** → DB 락 30s + cascade.
3. **event_id UNIQUE 없음** → 같은 event 2번 발송.
4. **SKIP LOCKED 없음** → 2 worker 같은 row 처리.
5. **ShedLock 없이 multi-instance** → 같은 batch query.
6. **pickup tx 길게** → 외부 호출까지 포함 → row lock 30s.
7. **stuck row reaper 없음** → worker crash 시 PROCESSING 영구.
8. **SENT 영구 보관** → outbox 백만 row → query 느림.
9. **partial index 없음** → worker query slow.
10. **Transient / Permanent 분류 X** → 영구 실패도 5회 retry.
11. **DLQ 알림 X** → admin 모름.
12. **delivery_results 안 저장** → per-channel 추적 X.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/outbox-pattern]]
- [[../implementation/outbox-worker-impl]]
