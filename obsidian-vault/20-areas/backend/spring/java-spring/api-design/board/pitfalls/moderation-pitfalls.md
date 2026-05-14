---
title: "Moderation / Block 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:28:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - pitfalls
  - moderation
---

# Moderation / Block 함정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — 작성자 검증 누락 (PATCH/DELETE)
다른 user 글 수정 가능.
→ @PreAuthorize.

## 함정 2 — DELETED 글 200 응답
soft delete 후 조회 200.
→ 404.

## 함정 3 — HIDDEN 작성자 못 봄
admin 모더 후 작성자 항의.
→ 작성자 + admin 은 access.

## 함정 4 — 자동 hide idempotency 없음
race 시 2번 hide.
→ status 검증.

## 함정 5 — 임계값 너무 낮음 (3)
신고 bombing.
→ 5 + 가중치.

## 함정 6 — 같은 user 중복 신고 허용
1 user 가 5회 = 자동 hide.
→ UNIQUE.

## 함정 7 — admin action audit 누락
admin abuse 추적 X.
→ moderation_audit_log.

## 함정 8 — Block self
A 가 A 차단 → 모든 글 안 보임.
→ CHECK 제약.

## 함정 9 — Block 한도 없음
무한 차단.
→ 1000 max.

## 함정 10 — Block filter 누락
차단 후 옛 글 계속 봄.
→ 모든 list 에 적용.

## 함정 11 — Admin 도 block filter
모더 시 차단 user 글 안 보임.
→ admin bypass.

## 함정 12 — Block cache 갱신 X
차단 후 옛 cache 의 글 봄.
→ AFTER_COMMIT DEL.

## 함정 13 — appeal 흐름 없음
정상 글 잘못 삭제 시 사용자 권리 X.
→ 7일 appeal.

---

## 관련

- [[../design-decisions/moderation-policy]] · [[../design-decisions/block-policy]]
- [[../security/moderation-impl]] · [[../security/block-filter]]
- [[pitfalls|↑ hub]]
