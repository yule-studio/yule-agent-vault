---
title: "task-log — issue #48 Harness 팀 아키텍처 도입"
kind: task-log
project: yule-studio-agent
issue: "#48"
branch: feature/agent-company-harness-48
worktree: ../yule-studio-agent-worktrees/48-harness
started_at: "2026-05-08"
author: engineering-agent/tech-lead
status: in_progress
contract: task-log/v0
---

# task-log — issue #48 Harness 팀 아키텍처 도입

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-08 | engineering-agent/tech-lead | 최초 시작 — kickoff, 정책 문서 + smoke 테스트 + vault 노트 3 종 작성 |

## 목표

issue #48 의 Harness 팀 아키텍처 팩토리 관점을 정리해, Yule engineering-gateway / role system / orchestration 구조를 "회사형 시니어 개발팀" 으로 강화하는 작업의 진행 로그를 한 곳에 둔다. 이 노트는 운영자 / 후속 PR 작성자 / cto-agent (미래) 가 본 작업의 의도와 결과를 1 분 안에 따라잡을 수 있게 하는 단일 진입점 역할을 한다.

## 현재 Yule 기준선 (확인된 출처)

- `README.md` / `docs/engineering.md` / `docs/github-agent-workos.md` — Discord intake / lifecycle / coding authorization / GitHub WorkOS surface.
- `policies/runtime/agents/engineering-agent/lifecycle-mvp.md` — 13 단계 + persistence + work_report gate + Obsidian write gate.
- `policies/runtime/agents/engineering-agent/role-profiles.md` — 7 RoleProfile + ParticipationLevel + fallback 6 + TechLeadAggregator.
- `policies/runtime/agents/engineering-agent/team-structure.md` — department gateway 모델 + cto-agent 도입 시점의 외부 인터페이스 이양.
- `policies/runtime/agents/engineering-agent/version-control.md` + `policies/reference/BRANCH_STRATEGY.md` (main-based, hotfix 동일 흐름) + `policies/reference/COMMIT_CONVENTION.md` (gitmoji + 3-section 한국어 본문).
- `.github/ISSUE_TEMPLATE/-feature--issue-template.md` — feature issue 본문 표준.
- 기존 G1~G6 PR (이미 main 합류) + engineering_intelligence 패키지 — Yule 의 GitHub WorkOS / RAG-CAG knowledge 흐름.

## 참고한 외부 레퍼런스

- 이슈 본문에 명시된 6 패턴 (Pipeline / Fan-out·Fan-in / Expert Pool / Producer-Reviewer / Supervisor / Hierarchical Delegation).
- `https://github.com/revfactory/harness` — README 직접 fetch 는 sandbox 에서 차단. 본 작업은 사용자 명시 패턴 명세 + agent-systems 일반 지식 + Yule 현 baseline 만 출처로 사용.

## 도입한 부분 (이번 세션 산출물)

| # | 산출물 | 위치 |
| --- | --- | --- |
| 1 | 정책 문서 — 6 패턴 매핑 + gateway 책임 + 역할 분리 + orchestration contract + skill disclosure + evolution loop + SPOF + next action | `policies/runtime/agents/engineering-agent/team-architecture-patterns.md` |
| 2 | smoke 테스트 — 정책 문서의 11 섹션 + 6 패턴 keyword + tech-lead 단일 write subject + safety rail 핵심 문구 | `tests/engineering/test_team_architecture_patterns_doc.py` |
| 3 | research note (본 작업의 근거 자료) | `10-projects/yule-studio-agent/research/2026-05-08_research-harness-team-patterns.md` |
| 4 | decision note (tech-lead 단일 write 주체 결정) | `10-projects/yule-studio-agent/decisions/2026-05-08_decision-tech-lead-single-write-subject.md` |
| 5 | task-log note (본 파일) | `10-projects/yule-studio-agent/task-logs/2026-05-08_task-log-issue-48-harness.md` |
| 6 | issue #48 kickoff comment | https://github.com/yule-studio/yule-studio-agent/issues/48#issuecomment-4404823515 |
| 7 | (예정) issue #48 progress comment + draft PR | 본 task 종료 직전 |

코드 변경 0 — 정책 + 테스트 (smoke) + vault 노트만.

## 보류 / 비도입 부분

- 코드 차원의 selector / dispatcher / lifecycle stage helper 변경 — `#25 선행 의존`.
- skill / hook / MCP server 파일 시스템 도입 — `#25 선행 의존`.
- 2 단 hierarchical delegation (sub-team / coordinator-agent / orchestrator-agent) — 1 단 모델 안정화 후.
- frontend / devops / qa / ai / product-designer 의 v1 contract 강화 — role-profiles.md 7 단계 절차로 후속 PR.
- `run_supervisor_watch_loop` 의 self-improvement 3 인자 wiring — M13 readiness §2 G-M12-01 의 후속 PR.

## 왜 실제 시니어 개발팀형 회사 구현에 필요한가

본 작업은 "코드 한 줄 안 바꾸고 정책만 바꾸는" 작업처럼 보이지만, **실제 회사형 운영의 약속을 종이에 박는 작업** 이다. Yule 은 이미 lifecycle 13 단계 / 7 RoleProfile / Coding authorization / GitHub WorkOS / engineering-intelligence / supervisor self-improvement 까지 갖췄지만, 이 부품들이 어떤 회사 모델을 상정하는지의 단일 진입점이 비어 있었다. 본 정책이 그 진입점을 채우면:

- 외부 surface 의 author / commenter / committer / vault author / Discord 회신 author 가 항상 tech-lead 1 명임이 명시된다 (책임 추적성).
- routing matrix / review gate / approval gate 가 한 표에 묶여 운영자가 1 분 안에 답한다 (운영 가용성).
- self-improvement signals + engineering-intelligence collector 가 "회사 retrospective 의 두 축" 으로 명시된다 (학습성).
- #25 / 다른 후속 PR 가 작업할 때 본 정책을 입력으로 받아 동일 회사 모델을 따른다 (확장성).

## 구현 / 설계 위치

- 정책 + smoke 테스트 = repo (`feature/agent-company-harness-48` worktree).
- 근거 자료 / 결정 / 진행 로그 = vault (`10-projects/yule-studio-agent/research|decisions|task-logs/`).
- 후속 PR 가 입력으로 받을 위치: 본 정책 §6 (routing matrix), §7 (skill disclosure), §11 (next action).

## 진행 단계 (이번 세션 timeline)

| 시각 | 액션 | 결과 |
| --- | --- | --- |
| 2026-05-08 — kickoff | git status / 브랜치 / 이슈 #48 확인 | clean main, 이슈 OPEN |
| 동상 | worktree 생성 | `../yule-studio-agent-worktrees/48-harness` (branch `feature/agent-company-harness-48`) |
| 동상 | 로컬 문서 11 종 읽기 | README / engineering / github-agent-workos / lifecycle-mvp / obsidian-memory / role-profiles / team-structure / version-control / BRANCH_STRATEGY / COMMIT_CONVENTION / issue template |
| 동상 | issue #48 kickoff comment | comment 4404823515 |
| 동상 | 정책 문서 작성 | `team-architecture-patterns.md` 11 섹션 |
| 동상 | smoke 테스트 작성 | `test_team_architecture_patterns_doc.py` |
| 동상 | vault 노트 3 종 작성 | research / decision / task-log |
| 2026-05-08 — verify | self-check — 신규 smoke + 인접 회귀 + 전체 회귀 | smoke 5 OK / `tests/engineering` 1040 OK / `tests/runtime` 252 OK / 전체 2824 OK · 0 FAIL · 1 skipped |
| (예정) | progress comment + commit + push + draft PR | (다음 단계) |

## 리스크 / 다음 액션

- 본 정책의 §6.1 routing matrix 와 `agents/lifecycle/role_selection.py` 의 fallback 정책이 drift 할 수 있다. 후속 PR 가 자동 정합성 검사 단위 테스트를 추가.
- #25 의 일정에 따라 본 정책 §7 (progressive disclosure) 만 먼저 활용해도 되는지 재검토.
- tech-lead degrade 빈도 통계가 쌓이면 §9 SPOF 리스크의 fallback 정책 재정의.

다음 액션 (this task 종료 전):

1. `tests/engineering/test_team_architecture_patterns_doc.py` 와 `tests/engineering` 인접 회귀 실행 → 결과를 본 노트에 추가.
2. issue #48 progress comment 작성 (정책 / 테스트 / vault 경로 / 변경 요약).
3. 작업 commit 후 push 하고 draft PR 생성.
4. 최종 보고에 worktree 경로 / vault 경로 / 변경 요약 / 테스트 결과 / PR 링크 수록.
