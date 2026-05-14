---
title: "Counter 함정 — Redis / DB sync"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:22:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - pitfalls
  - counter
---

# Counter 함정 — Redis / DB sync

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — 매 view 마다 DB UPDATE
인기 글 lock 폭증.
→ Redis INCR + 1h batch.

## 함정 2 — 트랜잭션 안 Redis INCR
rollback 시 Redis 만 +1.
→ AFTER_COMMIT.

## 함정 3 — DataIntegrityViolationException 안 catch
중복 좋아요 시 500.
→ silent catch.

## 함정 4 — Counter < 0
race / batch fail 시 음수.
→ DB CHECK + `Math.max(0, ...)`.

## 함정 5 — Batch sync 다중 실행
ShedLock 없으면 같은 cron 시점 다중.
→ ShedLock.

## 함정 6 — Redis key 영구 누적
삭제된 post 의 counter 영구.
→ post 삭제 시 Redis DEL.

## 함정 7 — Counter sync 너무 자주 (1min)
DB UPDATE 부담 ↑.
→ 1h.

## 함정 8 — Bot view count
크롤러 카운트 → ranking 왜곡.
→ UA filter.

## 함정 9 — `incrementView` 가 도메인 event 발행
매 view 마다 event → listener 폭증.
→ 도메인 event X (직접 incr).

## 함정 10 — view dedup 없음
새로고침 = 매번 +1.
→ IP+post 1분 dedup.

---

## 관련

- [[../design-decisions/like-counter]] · [[../design-decisions/view-counter]]
- [[pitfalls|↑ hub]]
