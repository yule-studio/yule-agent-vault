---
title: "MOC — Engineering Knowledge Surface (#69)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - engineering-knowledge
  - share-scope
  - issue-69
related:
  - _moc.md
---

# MOC — Engineering Knowledge Surface

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

issue #69 의 후속 — engineering knowledge 가 PUBLIC / TEAM_INTERNAL /
RESTRICTED 어느 surface 까지 노출 가능한지의 contract + retrieval
강화.

**[[_moc|↑ MOC 인덱스]]**

## 결정 / 리서치 / 작업 로그

- [[../decisions/decision-engineering-knowledge-share-boundary]] —
  share_scope enum (PUBLIC/TEAM_INTERNAL/RESTRICTED) + 8 결정 박스
- [[../research/research-engineering-knowledge-surface-strengthening]] —
  surface 강화 방안 리서치
- [[../task-logs/task-log-company-runtime-knowledge-surface]] — runtime
  통합 작업

## 핵심 contract

- `engineering_intelligence.models.KnowledgeShareScope` enum
- `obsidian.shareable_external_payload(item)` 단일 외부 진입점
- `discussion.context_pack.ContextPack.format_knowledge_evidence_block`
  evidence 블록

## 관련 hub

- [[plugins-catalog]] — paste-guard / claude-mem 가 본 정책 영향
