---
title: "사이클 별 토큰 / 비용 추적"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# 사이클 별 토큰 / 비용 추적

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

각 사이클 (F15 등) 별로 LLM 호출 token / cost / latency 를 sqlite 에 누적 + Discord `#봇-상태` 채널에 일일 요약.

## 동기

현재 운영자는 cycle 비용을 추정 못 함. 비싼 사이클을 식별하려면 자동 누적 필요.

## 구현 후보

`hookify` 의 COMPLETION hook 에 token count + cost calculator 추가. cycle id 는 git branch / PR 제목에서 추출.

## 성공 신호

매일 09:00 KST `#봇-상태` 에 "어제 토큰 NNk / $X.XX (cycle: f15)" 메시지.

## 블로커

Claude/Codex/Gemini 각 provider 의 token / cost 가 응답에 포함되는지 확인 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환