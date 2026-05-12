---
title: "tech-ops-ideas — 운영 / 기술 개선 아이디어"
kind: knowledge
project: agent-inbox
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T23:30:00+09:00
tags:
  - tech-ops-idea
  - inbox
  - improvement
---

# tech-ops-ideas — 운영 / 기술 개선 아이디어

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 10 아이디어 |

> yule-studio-agent 운영 / 코드 / 자동화의 **기술적 노하우 / 개선
> 아이디어**. 수익화가 아니라 **운영 효율 / 시스템 진화** 가 목표.

**[[../quick-notes|↑ quick-notes/]]**

## 현재 아이디어 (10)

- [[idea-pr-auto-label-by-scope]] — PR 자동 라벨 (scope 매트릭스)
- [[idea-vault-graph-clustering]] — Obsidian 그래프 hub 색 자동
- [[idea-cost-tracker-per-cycle]] — 사이클 별 토큰/cost 누적
- [[idea-discord-status-board]] — `#봇-상태` rich embed 매시간
- [[idea-mistake-ledger-dashboard]] — mistake 빈도 top 10 시각화
- [[idea-vault-search-cli]] — `yule vault search` 전 vault 검색
- [[idea-skill-codegen-from-prompts]] — 프롬프트 → Python skill codegen
- [[idea-cron-mvp-tasks]] — `yule cron` 정기 작업 단일 진입점
- [[idea-notion-incremental-sync]] — Notion incremental sync
- [[idea-agent-self-improvement-loop]] — agent prompt 자동 개선 loop

## 노트 표준 구조

각 아이디어 = 5 섹션:

1. **아이디어** — 무엇을
2. **동기** — 왜 (현재 문제)
3. **구현 후보** — 어떻게 (대략적 기술 스택)
4. **성공 신호** — 어떻게 알아볼지
5. **블로커** — 무엇이 막힐 수 있는지

## 운영 규칙

1. 아이디어는 idea. 채택 결정은 `decisions/` 에 별도 노트 + status `adopted`.
2. 채택 안 됨도 status `rejected` + 이유 본문 추가.
3. 1 개월 이상 unclaimed 아이디어는 `90-archive/deprecated-notes/` 로 검토.
4. 아이디어가 실제 작업이 되면 `task-logs/` 로 변환 + 본 노트 status `done`.

## 관련

- [[../project-ideas/project-ideas|↗ project-ideas]] — 수익화 프로젝트
  아이디어 (본 운영 아이디어와 분리)
