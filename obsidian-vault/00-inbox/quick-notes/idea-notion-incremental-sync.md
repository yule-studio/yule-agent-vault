---
title: "Notion incremental sync"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Notion incremental sync

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

이번에 한 Notion → vault 일괄 import 를 incremental 로 — Notion 의 last_edited_time 을 받아 변경분만 갱신.

## 동기

매번 19 페이지 전체 다시 가져오는 건 비효율. Notion API 의 search endpoint 로 변경분만 fetch.

## 구현 후보

`agents/notion/sync.py` 추가. cron 으로 daily 갱신. vault `10-projects/notion-import/` 자동 갱신.

## 성공 신호

Notion 페이지 수정 후 24h 안에 vault 도 갱신.

## 블로커

public page 의 last_edited_time 노출 여부 확인 필요. private 페이지면 Notion API token 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환