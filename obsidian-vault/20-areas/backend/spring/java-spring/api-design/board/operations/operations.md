---
title: "board §12 — 운영 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - operations
---

# board §12 — 운영 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[../board|↑ hub]]**  ·  ← [[../testing/testing]]  ·  → [[../implementation-order]]

> 배포 / 운영. signup operations 패턴 + board 특화 (S3 / Redis counter / 모더).

---

## 1. 노트

| 노트 | 무엇 |
| --- | --- |
| [[observability]] | board 특화 메트릭 (counter sync / 모더) |
| [[runbook]] | board 사고 대응 (XSS / spam / S3 cost) |

→ [[../../signup/operations/operations|↗ signup operations]] 의 configuration / deployment 그대로.

---

## 2. SLA / SLO

| 메트릭 | 목표 (p99) |
| --- | --- |
| 글 목록 조회 | < 200ms |
| 글 상세 조회 | < 100ms (cache hit) |
| 글 작성 | < 500ms |
| 댓글 작성 | < 200ms |
| 좋아요 toggle | < 100ms (Redis hit) |
| 검색 (FTS) | < 200ms (100만 posts) |
| 가용성 | 99.9% / 월 |

---

## 3. 시작 체크리스트

- [ ] signup operations 적용
- [ ] S3 bucket + CloudFront 설정
- [ ] Redis 7 (counter 용)
- [ ] FCM / APNs 인증서
- [ ] Batch job ShedLock 설정
  - [ ] post_like counter sync (1h)
  - [ ] post_view counter sync (1h)
  - [ ] hot_score 갱신 (1h)
  - [ ] tag usage_count (1h)
  - [ ] notification outbox cleanup (30d)
- [ ] S3 lifecycle (orphan 1d, 옛 attachment 90d archive)
- [ ] Admin dashboard (신고 / HIDDEN 글 review)
- [ ] alert — counter mismatch / outbox backlog / S3 cost

---

## 4. 관련

- [[../board|↑ hub]]
- [[../../signup/operations/operations|↗ signup operations]]
- [[observability]] · [[runbook]]
