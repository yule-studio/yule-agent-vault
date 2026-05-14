---
title: "board §13 — 함정 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - pitfalls
  - hub
---

# board §13 — 함정 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[../board|↑ hub]]**  ·  ← [[../implementation-order]]

> board 특화 함정. signup pitfalls 베이스 + board 카테고리.

---

## 1. 카테고리

| 노트 | 무엇 | 함정 수 |
| --- | --- | --- |
| [[counter-pitfalls]] | counter race / sync / Redis | 8+ |
| [[content-pitfalls]] | XSS / markdown / 첨부 | 10+ |
| [[pagination-pitfalls]] | cursor / sort / N+1 | 7+ |
| [[moderation-pitfalls]] | 신고 / 차단 / admin | 8+ |

→ signup 의 database/security/transaction/domain/operations pitfalls 는 cross-link 만.

---

## 2. 코드 리뷰 체크리스트

PR 마다:

### 2.1 Counter / 동시성

- [ ] 트랜잭션 안 Redis INCR X (AFTER_COMMIT)
- [ ] DataIntegrityViolationException catch (idempotent)
- [ ] decrement 의 `Math.max(0, ...)`
- [ ] @Version 충돌 retry 정책

### 2.2 Content

- [ ] XSS sanitize (rendering 시점)
- [ ] markdown 저장은 raw
- [ ] content / title 길이 검증
- [ ] 첨부 S3 exists 검증
- [ ] presigned 5분 TTL

### 2.3 Pagination

- [ ] cursor base64
- [ ] cursor type 검증
- [ ] limit max 50
- [ ] block filter 적용
- [ ] HIDDEN / DELETED 제외

### 2.4 모더 / 차단

- [ ] 작성자 검증 (PATCH/DELETE)
- [ ] AUTO_HIDDEN idempotency
- [ ] admin audit log
- [ ] block 한도 1000
- [ ] block cache invalidate

---

## 3. 관련

- [[../board|↑ hub]]
- [[../../signup/pitfalls/pitfalls|↗ signup pitfalls]]
- counter-pitfalls · content-pitfalls · pagination-pitfalls · moderation-pitfalls
