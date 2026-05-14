---
title: "Rate / spam 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:32:00+09:00
tags: [backend, java-spring, api-design, notification, pitfalls, rate]
---

# Rate / spam 함정

**[[pitfalls|↑ hub]]**

---

1. **rate limit 없음** → 사용자 분당 100 알림 → OFF → 중요 알림 무시.
2. **강제 ON 도 rate 적용** → 보안 alert drop.
3. **집계 없음** → "12명이 좋아요" 못 함 → 개별 N개 발송.
4. **dedup 없음** → 같은 event 중복.
5. **quiet hours 글로벌 timezone** → 미국 사용자가 새벽 알림.
6. **default 모두 ON** → 신규 사용자 spam.
7. **abuse 검출 없음** → 단일 사용자 1만 발송 인지 못함.
8. **SES bounce rate** 무시 → AWS account suspend.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/rate-limit]]
- [[../design-decisions/batch-aggregation]]
- [[../design-decisions/user-preferences]]
