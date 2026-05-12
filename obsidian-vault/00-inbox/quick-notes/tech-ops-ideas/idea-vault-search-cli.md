---
title: "Vault 검색 CLI (전체 코퍼스)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Vault 검색 CLI (전체 코퍼스)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

`yule vault search <query>` — vault 의 모든 노트 (28+) 를 frontmatter + 본문 기반으로 검색. claude-mem 의 메모리 인덱스 활용.

## 동기

Obsidian 검색은 vault 안에서만 작동. CLI 가 있으면 자동화 에이전트가 작업 시작 전 vault 컨텍스트 자동 surface.

## 구현 후보

`agents/obsidian/search.py` 추가 — frontmatter index + fts5 SQLite.

## 성공 신호

`yule vault search "manifest unification"` → 관련 5 노트 + 한 줄 발췌.

## 블로커

vault index 의 freshness — 새 노트 추가 시 자동 reindex 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환