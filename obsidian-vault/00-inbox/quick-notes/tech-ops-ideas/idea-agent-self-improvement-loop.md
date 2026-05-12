---
title: "Agent self-improvement loop"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Agent self-improvement loop

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

Mistake ledger + decision 노트 + task-log 를 입력으로 매 사이클 종료 시 agent prompt 자동 개선 제안.

## 동기

F10/F14 의 다음 단계. 현재는 mistake 가 ledger 에만 누적, prompt 변경은 수동.

## 구현 후보

`agents/self_improve/` — 5 소스 입력 + Claude 가 prompt diff 제안 + 운영자 승인 게이트.

## 성공 신호

매 사이클 종료 후 PR 1 개 자동 생성 — "prompt update for backend-engineer based on 3 mistakes".

## 블로커

자동 prompt 변경은 high-risk — F12 tool-call-gate 의 supervised 매트릭스 통과 필수.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환