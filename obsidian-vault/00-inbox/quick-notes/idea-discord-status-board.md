---
title: "Discord 상태 보드 (rich embed)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Discord 상태 보드 (rich embed)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

`#봇-상태` 채널에 매시간 rich embed 1 개 — 활성 워커 수 / 대기 큐 깊이 / 최근 mistake / 진행중 PR 링크.

## 동기

현재 supervisor 진단은 CLI 출력만. 운영자가 Discord 만 보고 시스템 상태 한 눈에.

## 구현 후보

`yule supervisor run` 결과를 Discord embed 로 매핑. F11 의 `discussion-response` 플러그인 확장.

## 성공 신호

Discord 모바일에서 embed 카드 1 장으로 시스템 상태 식별.

## 블로커

sandbox 차단 환경에서는 Discord 미연결 — 사용자 호스트 검증 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환