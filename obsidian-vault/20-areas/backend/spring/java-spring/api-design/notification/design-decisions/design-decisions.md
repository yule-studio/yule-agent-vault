---
title: "notification design-decisions hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:18:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
---

# notification design-decisions hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../notification|↑ hub]]**

---

## 1. 결정 목록

| # | 결정 | 노트 |
| --- | --- | --- |
| 1 | **채널 선택** ★ | [[channel-selection]] |
| 2 | **Outbox 패턴** ★ | [[outbox-pattern]] |
| 3 | 사용자 preference | [[user-preferences]] |
| 4 | **Device 관리** ★ | [[device-management]] |
| 5 | retry 전략 | [[retry-strategy]] |
| 6 | dedup 전략 | [[dedup-strategy]] |
| 7 | rate limit | [[rate-limit]] |
| 8 | template / i18n | [[template-i18n]] |
| 9 | batch aggregation | [[batch-aggregation]] |
| 10 | ordering | [[ordering]] |
| 11 | **Kafka 고도화** ★ (F8+) | [[kafka-event-driven]] |

---

## 2. 4구조

모든 결정 노트는 동일 패턴:

1. 본 vault 결정
2. 왜 필요 / 안 하면 / 대안 / 트레이드오프
3. 구현 (코드)
4. 함정
5. 다른 컨텍스트

---

## 3. 관련

- [[../notification|↑ hub]]
- [[../signup/design-decisions/design-decisions|↗ signup]]
- [[../product/design-decisions/design-decisions|↗ product]]
