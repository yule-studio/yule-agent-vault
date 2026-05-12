---
title: "Hermes Agent → Yule Studio Agent 흡수 분석 (issue #59) — task-log"
topic: hermes-yule-integration-59
kind: task-log
project: yule-studio-agent
session_id: issue-59
issue_number: 59
issue_url: https://github.com/yule-studio/yule-studio-agent/issues/59
external_reference: https://github.com/NousResearch/hermes-agent
actor_role: engineering-agent/tech-lead
status: phase_1_completed
created_at: 2026-05-08
agent: engineering-agent/tech-lead
---

# Hermes Agent → Yule Studio Agent 흡수 분석 (issue #59) — task-log

본 노트는 issue #59 진행 중 운영자 가시성 확보를 위한 작업 로그다. Obsidian vault 의 다른 산출물(research / decision note) 과 GitHub issue / draft PR 의 cross-reference 가 한 곳에서 보이도록 한다.

## 목표

Hermes Agent 의 messaging gateway / persistent memory / cross-session recall / context compression / skill self-improvement / scheduled automations / subagent orchestration / workspace migration 중 Yule 의 "실제로 일하는 시니어 개발팀형 에이전트" 구현에 필요한 요소를 선별·반영한다. 단순 모방이 아니라 **diff 기반** — 현재 Yule 구조를 기준선으로 두고 Hermes 가 더 나은 부분만 흡수한다.

## 현재 Yule 기준선 (kickoff 시점)

- **Discord gateway** = `discord/engineering_channel_router.py` (라우팅) + `discord/engineering_team_runtime.py` (member-bot turn). `decide_routing()` 이 4 가지 action(`join_existing_work` / `create_new_work` / `ask_for_clarification` / `append_context_only`) 중 하나로 라우팅.
- **persistent memory** = SQLite (`.cache/yule/cache.sqlite3`) + Obsidian vault. `agents/lifecycle_persistence.py` 가 `session.extra` 영속화, `obsidian_export.py` 가 결정적 export.
- **role 시스템** = 7 role (tech-lead 포함) + `ParticipationLevel` 5 단계(required/primary/reviewer/optional/excluded). 셀렉터는 `role_profiles_data.py` 단일 source of truth.
- **research collector** = `auto_collect_or_request_more_input()` 가 active role 별 coverage 평가, multi-provider(Tavily/Brave) + 예산 / dedup / no-progress 안전 종료.
- **work_report lifecycle gate** = `agents/lifecycle_status.compute_report_status()` 가 INSUFFICIENT / INTERIM / READY / FINAL 결정. Obsidian write gate 도 같은 source.
- **knowledge note** = `agents/knowledge_writer.py` (M9/M10 phase C). 단순 source dump 가 아니라 작업 목적/결론/handoff 가 들어간 사람 가독 문서.
- **GitHub WorkOS (G1~G6)** — issue triage(senior_triage) → branch/PR plan → draft PR 까지. L0/L1/L2 까지 자동, **L3+ 는 항상 사람 게이트**.
- **role-runner dispatch (M11b)** = `set_role_runner_dispatch()` 로 게이트웨이가 Claude/Codex/Ollama → deterministic 우선순위 chain. 기본 deterministic, env opt-in 으로 LLM 활성.
- **scheduling** = 부분적 — `daily_briefing` / `daily_preparation` / `checkpoint_notification` 정도. 일반화된 cron 흐름은 없음.
- **subagent / parallel work** = member-bot 7 개가 독립 프로세스 + role-scoped research. 작업 1 건 안에서의 parallel write 는 명시적으로 거부 (single executor 정책).

## 분석 대상 (Hermes 기준 8 영역)

| 영역 | 현재 Yule 의 자리 | 비교 진행 |
|---|---|---|
| messaging gateway | `discord/engineering_channel_router.py` | **이번 분석 핵심** |
| persistent memory | SQLite + Obsidian | 이번 분석 핵심 |
| cross-session recall | recall via thread_id / topic_ledger | 이번 분석 핵심 |
| context compression | knowledge writer + research budget | 이번 분석 핵심 |
| skill self-improvement | `lifecycle/self_improvement.py` skeleton | 이번 분석 핵심 |
| scheduled automations | daily briefing + checkpoint loop | 이번 분석 핵심 |
| subagent / parallel | member-bot × 7 + role-scoped | 이번 분석 핵심 |
| workspace / context migration | session.extra round-trip | 이번 분석 보조 |

## 참고한 외부 레퍼런스

- https://github.com/NousResearch/hermes-agent — README / docs / 모듈 구조. 추가 자료 수집 결과는 research note 에 정리.

## 도입한 부분 / 보류한 부분 / 비도입

(decision note 와 함께 수정. 분석 진행 중에 단계적으로 채움)

## 왜 실제 시니어 개발팀형 회사 구현에 필요한지

- **memory**: 한 사람이 모든 과거 결정을 다시 떠올리는 비용 = 0 이어야 함. Hermes 의 memory 모델은 단발성 chat 이 아니라 작업 세션 간 reusable knowledge 로 취급. Yule 에서 동일 효과를 내려면 recall trigger / compression policy / skill capture 가 명확해야 한다.
- **scheduled automation**: 시니어 개발자는 "월요일 정기 점검" 같은 반복 작업을 자동화하지 않으면 시간이 갉힘. 단발 cron 이 아니라 lifecycle 과 결합된 scheduled work 로 운영해야 한다.
- **self-improvement / skill capture**: 같은 실수를 두 번 하지 않게 하려면 회고를 자산화해야 함. Hermes 의 skill evolution 흐름이 단순 prompt template 보강이 아닌 운영 메모리 자산 운용이라면 그 부분만 Yule 에 옮길 가치가 있음.

## 구현 위치 후보

- 코드 변경 영역: `policies/` (정책 강화), `agents/lifecycle/` (helper 보강), `agents/runners/` (필요 시 dispatch 보강).
- **이번 phase 는 정책 + 문서 + 최소 helper** 까지로 한정. 큰 코드 변경은 별도 phase.

## 리스크와 다음 액션

- 리스크 1: **#25 / #48 와 영역 겹침**. 공통 변경(team-structure / lifecycle 본문) 은 손대지 않고 별도 정책으로 분리.
- 리스크 2: **외부 레퍼런스 무비판 흡수**. Hermes 가 multi-LLM general-purpose agent 인 반면 Yule 은 개인 홈서버 single-operator 환경. 비대칭 영역(예: full multi-tenant memory) 은 적용 안 함.
- 리스크 3: **secret 범위 확장**. scheduled automation 이 외부 트리거를 넓히면 token 노출 표면이 늘어남 — env / .env.local 정책 강화 필요.

다음 액션:
1. Hermes README/docs 분석 → mapping table 채우기
2. research note 본문 작성
3. decision note 본문 작성
4. 정책 보강 commit (Hermes 흡수 항목)
5. progress comment + draft PR open

## 작업 결정 기록

| 시각 | 결정 |
|---|---|
| 2026-05-08 kickoff | engineering-agent/tech-lead 단독 executor / writer. 다른 6 role 은 분석 관점으로만 인용. |
| 2026-05-08 kickoff | branch `feature/agent-company-hermes-59`, worktree `../yule-studio-agent-worktrees/59-hermes` 신규 생성. base = main HEAD `55e8b3d`. |
| 2026-05-08 kickoff | #25 / #48 영역과 겹치는 공통 모듈은 손대지 않는다. Hermes 고유의 memory / gateway / evolution / automation 관점에 집중. |
| 2026-05-08 분석 완료 | Hermes 8 영역 분석 결과 — 5 부분 도입 (D-2 memory / D-3 recall / D-4 compression / D-5 self-improvement / D-6 scheduled), 3 비도입 (D-1 messaging gateway / D-7 subagent / D-8 workspace migration). |
| 2026-05-08 phase 1 완료 | 정책 5 종 + agent.json 등록 (15→20 entries). 코드 변경 0 건. 71 테스트 OK. draft PR #67. |
| 2026-05-08 phase 1 완료 | 후속 milestone 각 정책에 P1/P2/P3 우선순위로 명시 — retrieval boost wiring (P1), `yule insights` CLI (P1), role-runner dispatch input 압축 (P1), 회고 후보 신호 (P1), scheduler 코드 (P1). |

## Phase 1 산출물 링크

- **branch**: `feature/agent-company-hermes-59`
- **worktree**: `/Users/masterway/local-dev/yule-studio-agent-worktrees/59-hermes`
- **commit**: `b0e638e ✨ Hermes Agent 흡수 — memory/recall/compression/self-improvement/scheduled-automation 정책 5종`
- **draft PR**: https://github.com/yule-studio/yule-studio-agent/pull/67
- **issue progress comment**: https://github.com/yule-studio/yule-studio-agent/issues/59#issuecomment-4404937467
- **issue kickoff comment**: https://github.com/yule-studio/yule-studio-agent/issues/59#issuecomment-4404821267

## 다음 phase (issue #59 후속)

- 각 정책의 P1 milestone 별로 별도 issue 생성:
  - memory-policy retrieval boost wiring
  - recall-policy `yule insights` CLI
  - context-compression role-runner dispatch input 압축
  - self-improvement-flow lifecycle 회고 후보 신호
  - scheduled-automation scheduler 코드 + CLI
- draft PR #67 검토 → 사용자 승인 → 일반 merge.
