---
title: "quick-notes — 떠오른 아이디어 / 메모"
kind: knowledge
project: agent-inbox
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - inbox
  - quick-notes
  - ideas
---

# quick-notes — 떠오른 아이디어 / 메모

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 10 아이디어 |

**[[../inbox|↑ 00-inbox/]]**

## 이 폴더가 묶는 것

운영자 / 자동화 에이전트가 작업하다 떠올린 **아이디어 / 후속 작업 후보**.
공식 결정이나 작업 로그 전 단계 — 채택되면 `_moc/` 또는 `task-logs/`
로 이주.

## 현재 아이디어 (10)

- [[idea-pr-auto-label-by-scope]] — PR 라벨 자동 부착 (scope 기반)
- [[idea-vault-graph-clustering]] — Obsidian 그래프 hub 별 색 자동
- [[idea-cost-tracker-per-cycle]] — 사이클 별 token/cost/latency 누적
- [[idea-discord-status-board]] — `#봇-상태` rich embed 매시간
- [[idea-mistake-ledger-dashboard]] — mistake 빈도 top 10 시각화
- [[idea-vault-search-cli]] — `yule vault search` 전 vault 검색
- [[idea-skill-codegen-from-prompts]] — 프롬프트 → Python skill codegen
- [[idea-cron-mvp-tasks]] — `yule cron` 정기 작업 단일 진입점
- [[idea-notion-incremental-sync]] — Notion incremental sync
- [[idea-agent-self-improvement-loop]] — agent prompt 자동 개선 loop

## 운영 규칙

1. **아이디어 ≠ 결정**. 본 폴더에 노트가 있다고 즉시 채택되는 것은 아님.
2. 채택 결정은 `decisions/` 에 별도 노트 + 본 노트의 status `adopted`
   갱신.
3. 채택 안 됨 (rejected) 도 status 갱신 + 이유 본문 추가.
4. 1 개월 이상 unclaimed 아이디어는 `90-archive/deprecated-notes/` 로
   이주 검토.
