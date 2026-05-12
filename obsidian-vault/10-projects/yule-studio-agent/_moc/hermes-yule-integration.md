---
title: "MOC — Hermes Agent → Yule 흡수 (#59)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: completed
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - hermes-yule
  - issue-59
related:
  - _moc.md
---

# MOC — Hermes → Yule 흡수

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

NousResearch Hermes agent 의 회사형 팀 패턴을 Yule engineering-agent
로 흡수한 사이클 (issue #59) 의 모든 노트.

**[[_moc|↑ MOC 인덱스]]**

## 결정 / 리서치 / 작업 로그

- [[../decisions/decision-hermes-yule-integration]] — Hermes 패턴
  흡수 결정
- [[../research/research-hermes-agent-architecture-deep-dive]] —
  Hermes 아키텍처 deep-dive (입력)
- [[../research/research-harness-team-patterns]] — Harness 회사형 팀
  패턴 비교 (보조 입력)
- [[../task-logs/task-log-hermes-tech-lead-issue-59]] — 실제 흡수 작업

## 핵심 흡수 패턴

- Producer-Reviewer 분리 (executor 1 + reviewer N)
- Hierarchical Delegation (tech-lead → 6 역할)
- Single-author external surface (tech-lead 1 명만 외부 발화)

## 관련 hub

- [[issue-73-tech-lead-runtime]] — Hermes 패턴 위에 올라간 runtime loop
- [[f15-corporate-structure]] — 본 사이클이 만든 6 역할 골격이 F15
  에서 부서 단위 (CTO / CPO / CMO) 로 확장
