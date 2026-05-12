---
title: "프롬프트 → skill 자동 codegen"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# 프롬프트 → skill 자동 codegen

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

`prompts/skills/<role>/*.md` 의 프롬프트 템플릿을 Python 함수로 자동 codegen — `from yule.skills.engineering import research_pack`.

## 동기

F14 의 일부 시도. 현재 프롬프트는 문자열 템플릿. codegen 으로 가면 IDE autocomplete + 타입 안전.

## 구현 후보

`agents/skills/codegen.py` — frontmatter 의 input/output schema 파싱 → dataclass.

## 성공 신호

새 skill 추가 시 `from yule.skills.<area>.<name>` 자동 등장.

## 블로커

F14 의 cache invalidation 정책과 sync 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환