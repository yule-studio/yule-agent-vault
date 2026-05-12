---
title: "F15 #126 — Corporate Structure (CTO/CPO/CMO 부서 골격)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-11T00:00:00+09:00
tags:
  - moc
  - hub
  - corporate-structure
  - issue-126
  - f15
related:
  - _moc.md
---

# 이 hub 가 묶는 것

F15 사이클의 모든 결정 · 정책 · 작업 로그를 한 곳에 모은다. 사이클이
끝나면 다음 진입자 (또는 미래의 나) 가 여기서 시작.

## 결정 노트

- [[decision-product-vs-marketing-cpo-cmo-separation-issue-126]] —
  product-agent (CPO) vs marketing-agent (CMO) 의 7 결정 박스

## 작업 로그

- [[task-log-corporate-structure-f15-issue-126]] — F15
  commit 1~8 진행
- [[task-log-f15-smoke-test-non-discord]] — Discord 없는
  e2e smoke (session 7c922f0a8e6d)
- [[2026-05-12_task-log_vault-sweep-graph-consolidation]] — 본 vault
  정리 task-log

## 관련 hub

- [[manifest-migration]] — F15 의 핵심 작업이었던 agent.json →
  manifest.json 통일
- [[plugins-catalog]] — F15 commit 7 에서 작성된 11 플러그인 카탈로그

## repo 안 참조

- `policies/runtime/agents/corporate-org-chart.md` — C-level 매트릭스
- `policies/runtime/agents/department-boundary-policy.md` — 부서 경계
- `agents/product-agent/` — 3 역할 (product-manager / user-researcher /
  growth-analyst)
- `agents/marketing-agent/` — 4 역할 (growth-marketer /
  content-strategist / seo-specialist / brand-manager)
- `agents/engineering-agent/` — 7 역할 + manifest 통일 완료

## 사이클 외 잔여

- hr-agent / finance-agent / sales-cs-agent / legal-agent 골격 (아직 미구현)
- PM skills catalog (pm-skills 참조)
- F15 governance 테스트
