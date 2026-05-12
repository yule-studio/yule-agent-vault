---
title: "PR 자동 라벨 by scope 매트릭스"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# PR 자동 라벨 by scope 매트릭스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

PR title / branch prefix / changed files 를 보고 scope 라벨 (f15 / discord / vault / plugins / etc.) 자동 부착. 운영자가 GitHub UI 에서 한 눈에 어떤 사이클의 PR 인지 식별.

## 동기

현재는 PR 라벨이 수동. F15 같은 큰 사이클에서 commit 10+ 개의 PR 이 라벨 없으면 history grep 이 무거움.

## 구현 후보

`.github/workflows/auto-label.yml` + `actions/github-script`. F12 tool-call-gate 와 무관 (GitHub Actions surface).

## 성공 신호

다음 사이클 시작 시 PR 제목만으로 라벨 자동 부착 + Discord notify 메시지에 라벨 표시.

## 블로커

GitHub Actions 의 GITHUB_TOKEN scope (issues:write 필요). 본 레포의 token permissions 정책 확인.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환