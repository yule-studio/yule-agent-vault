---
title: "Mistake ledger 시각화"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Mistake ledger 시각화

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

`hookify` mistake_ledger 의 누적 signature 를 시각화 — 가장 자주 발생한 mistake top 10 + 발생 빈도 추이.

## 동기

ledger 가 grep 으로만 조회 가능. 운영자가 패턴을 한 눈에 보면 정책 갱신 우선순위가 명확.

## 구현 후보

`yule supervisor mistake-stats` CLI 추가. JSON 출력 → 사용자 vault `60-troubleshooting/` 자동 동기화.

## 성공 신호

`obsidian.filename.date-prefix` 같은 mistake 가 정책 갱신 후 빈도 0 으로.

## 블로커

시각화는 CLI 출력 또는 markdown 표로 충분 (별도 dashboard UI 도입 미정).

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환