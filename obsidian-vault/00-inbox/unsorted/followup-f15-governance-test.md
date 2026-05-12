---
title: "F15 governance 테스트 후속"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# F15 governance 테스트 후속

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 잔여 작업

F15 사이클의 governance 테스트 (manifest ↔ 정책 ↔ vault 일관성) 가 아직 미구현.

## 범위

`tests/engineering/test_corporate_structure_governance.py` — corporate-org-chart.md 의 부서 매트릭스가 실제 `agents/<dept>/manifest.json` 과 일치 검증.

## 우선순위

PR 머지 전 회귀 게이트로 들어가야 함. 다음 사이클 commit 1 에서 진행.

## 영향

부서 추가 시 governance 가드가 자동 검증 → 운영자가 일관성 깨질 위험 차단.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환