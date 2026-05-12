---
title: "Cron MVP — 정기 작업 등록"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Cron MVP — 정기 작업 등록

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

`yule cron add "daily 09:00 yule daily plan"` — 정기 작업의 단일 진입점. F13 digest scheduler 의 일반화.

## 동기

F13 은 부서 자료 수집 한정. 다른 정기 작업 (mistake ledger sweep / supervisor diagnostic / vault auto push) 도 한 곳에서 관리.

## 구현 후보

launchd plist 자동 생성 / systemd timer / 또는 in-process scheduler.

## 성공 신호

`yule cron list` → 5-10 정기 작업 + 다음 실행 시각.

## 블로커

sandbox 환경에서는 daemon 운영 불가 — 사용자 호스트 검증 필요.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환