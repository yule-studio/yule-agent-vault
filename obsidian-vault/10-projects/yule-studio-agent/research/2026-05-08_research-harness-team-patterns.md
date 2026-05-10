---
title: "Harness 6 팀 아키텍처 패턴 매핑 — Yule engineering-agent 연구 노트"
kind: research
project: yule-studio-agent
issue: "#48"
branch: feature/agent-company-harness-48
collected_at: "2026-05-08"
author: engineering-agent/tech-lead
status: research
contract: research/v0
---

# Harness 6 팀 아키텍처 패턴 매핑 — Yule engineering-agent 연구 노트

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-08 | engineering-agent/tech-lead | 최초 수집 — issue #48 의 baseline 매핑 |

## 목표

Harness 의 팀 아키텍처 팩토리 관점과 6 가지 팀 패턴 (Pipeline / Fan-out·Fan-in / Expert Pool / Producer-Reviewer / Supervisor / Hierarchical Delegation) 을 Yule engineering-agent 의 현재 baseline 에 매핑해, 어떤 패턴을 **이미 갖고 있는지**, 어떤 패턴을 **부분 채택**, 어떤 패턴은 **명문화 필요** 인지 정리한다. 이 노트는 정책 문서 `policies/runtime/agents/engineering-agent/team-architecture-patterns.md` 의 근거 파일 역할을 한다.

## 현재 Yule 기준선 (출처)

- `docs/engineering.md` — Discord 라우팅 + research 자동 수집 + coding authorization MVP + Obsidian sync.
- `policies/runtime/agents/engineering-agent/lifecycle-mvp.md` — lifecycle 13 단계 + session.extra persistence policy + work_report gate + Obsidian write gate + typing policy.
- `policies/runtime/agents/engineering-agent/role-profiles.md` — 7 역할 RoleProfile + 5 단계 ParticipationLevel + fallback 6 종 + TechLeadAggregator 정책.
- `policies/runtime/agents/engineering-agent/team-structure.md` — department gateway 모델 + 회사 전체 agent platform 위치 + 멤버 책임 + LLM 백엔드 공유.
- `policies/runtime/agents/engineering-agent/version-control.md` + `BRANCH_STRATEGY.md` + `COMMIT_CONVENTION.md` — main-based 브랜치 + gitmoji + 한국어 3-section 본문.

## 참고한 외부 레퍼런스

- 이슈 본문에 명시된 6 패턴 명세 (Pipeline / Fan-out·Fan-in / Expert Pool / Producer-Reviewer / Supervisor / Hierarchical Delegation).
- `https://github.com/revfactory/harness` — README 직접 fetch 는 sandbox 정책상 차단되어 (gh API 권한 거부) 사용자 명시 명세 + agent-systems 일반 지식만 사용.
- agent-systems 일반 지식 (orchestration patterns, multi-agent dispatcher, supervisor trees, BFG fan-out/fan-in, LangGraph 류 대안) — 본 노트는 이를 정의 anchor 로만 사용하고 외부 인용은 두지 않음.

## 6 패턴 매핑 요약 (research 시점 결론)

| 패턴 | Yule 현재 대응 | 매핑 강도 | 핵심 근거 |
| --- | --- | --- | --- |
| Pipeline | lifecycle 13 단계 (`intake → … → coding_job_ready`) | **강 — 운영 중** | lifecycle-mvp.md §3 |
| Fan-out·Fan-in | `role_take` job fan-out → `tech_lead_aggregator.aggregate_role_outputs` fan-in | **강 — 운영 중** | tech_lead_aggregator.py + role-profiles.md §"TechLeadAggregator 정책" |
| Expert Pool | RoleSelector + 7 RoleProfile + activation_keywords + ParticipationLevel | **중 — 부분 채택** | role_selection.py + role-profiles.md §"Selector 동작" |
| Producer-Reviewer | Coding authorization MVP + GitHub WorkOS senior_triage `approval_required_actions` | **중 — 부분 채택** | docs/engineering.md "Coding Agent Authorization MVP", agents/github_workos/triage.py |
| Supervisor | `run_supervisor_watch_loop` + heartbeat + lease reaper + self-improvement detect+dispatch | **강 — 운영 중** (단 `_run_async` wiring 한 줄 미완) | M13 readiness §2 G-M12-01 |
| Hierarchical Delegation | `engineering-agent (department) → tech-lead → 6 역할 → executor` 1 단 계층 | **강 — 운영 중** | team-structure.md §"Position In Company-wide Agent Platform" |

## 도입한 부분 (정책 문서에 명문화)

1. **gateway 기본 패턴 조합 = Pipeline + Fan-out·Fan-in + Expert Pool + Producer-Reviewer + Supervisor + 1 단 Hierarchical Delegation.** 본 정책 §3.1.
2. **모든 외부 surface (Discord 회신 / GitHub issue/PR / Obsidian note / commit 메시지) 의 author = tech-lead 1 명** ("실행 주체") + 6 역할 = 분석 관점만. 본 정책 §5.
3. **Routing Matrix / Review Gate / Approval Gate** 를 본 정책 §6 에 표로 못박아, selector → 활성 역할 → write subject → review gate → approval gate 의 흐름을 한 화면에서 본다.
4. **Progressive disclosure skill 구조** 를 L0 (mission/activation/output_sections) / L1 (operating_modes/reasoning_flow/required_questions) / L2 (contract_standards) 3 단으로 정의. 본 정책 §7. 단, 실제 skill / hook / MCP wiring 은 #25 선행 의존.
5. **Harness evolution feedback loop** = self-improvement signals (부서 내부 회귀) + engineering-intelligence collector (부서 외부 학습) 양 축으로 정의. 본 정책 §8.

## 보류 / 비도입 부분

| 항목 | 이유 |
| --- | --- |
| 코드 차원의 selector / dispatcher / lifecycle stage helper 변경 | #25 와 직접 충돌. 본 이슈는 정책만 잡고 코드 변경 0 으로 둔다. |
| 2 단 hierarchical delegation (sub-team / coordinator-agent / orchestrator-agent) | 1 부서 + 6 역할 + 1 게이트웨이 모델이 안정화되기 전 sub-team 분기 금지. cto-agent 도입 시점에 재검토. |
| skill / hook / MCP 파일 시스템 도입 | #25 선행 의존 — 본 정책은 design intent 만 명시. |
| frontend / devops / qa / ai / product-designer 의 `agent.json` v1 contract 강화 | role-profiles.md §"다른 role 로 확장할 때 재사용할 패턴" 의 7 단계 절차를 그대로 따라 후속 PR 가 점진 진행. |
| Discord 메시지 라우팅 흐름 자체 (intake / decide_routing) | #25 가 잡는 영역. 본 정책은 책임 정의만 두고 호출 surface 는 손대지 않는다. |

## 왜 실제 시니어 개발팀형 회사 구현에 필요한가

- **단일 외부 author** 가 없으면 외부에서 보는 Yule 의 발화는 6 명이 동시에 떠드는 모양이 된다 — 실제 회사에서는 tech-lead 또는 PM 한 사람이 합의된 결론을 외부에 발표하고 6 관점은 내부 review 에서만 등장한다. 본 정책은 그 실제 회사의 운영을 그대로 정형화한다.
- **review gate / approval gate / routing matrix** 가 한 표에 묶여 있어야 운영자가 "지금 이 작업이 왜 사용자 승인을 기다리고 있나" 를 1 분 안에 답할 수 있다. lifecycle-mvp.md / role-profiles.md / coding-authorization 이 분리돼 있으면 답이 3 곳에 흩어져 같은 질문이 매번 다시 발생한다.
- **Harness evolution = self-improvement + engineering-intelligence** 양 축으로 정의하면 부서가 자기 자신을 개선하는 surface 가 명시된다. 단순 "운영 중 회귀를 잡자" 는 슬로건과 다르게, 두 surface 가 어디에 떨어지고 누가 검토하는지를 정책으로 정해 둘 수 있다.

## 구현 / 설계 위치

- **정책 문서**: `policies/runtime/agents/engineering-agent/team-architecture-patterns.md` (신규).
- **smoke 테스트**: `tests/engineering/test_team_architecture_patterns_doc.py` (정책 문서 존재 + 11 섹션 + 6 패턴 keyword + tech-lead 단일 write subject + main/release/force push 안전 rail).
- **vault 노트 3 종**:
  - `10-projects/yule-studio-agent/research/2026-05-08_research-harness-team-patterns.md` (본 파일).
  - `10-projects/yule-studio-agent/decisions/2026-05-08_decision-tech-lead-single-write-subject.md`.
  - `10-projects/yule-studio-agent/task-logs/2026-05-08_task-log-issue-48-harness.md`.
- **연관 surface 변경 0** — 코드 / 회귀 / 인덱스 변경 없음. 본 이슈는 정책 + 노트 + 테스트 + draft PR 까지가 deliverables 의 전부.

## 리스크 / 다음 액션

- **#25 와의 책임 경계 모호 가능성** — #25 가 actual skill/hook/MCP 를 추가할 때 본 정책 §7 (progressive disclosure) 을 입력으로 받을지 다시 정의할지 결정해야 한다. 본 정책은 "#25 선행 의존" 으로 명시했지만, 만약 #25 가 일정상 늦어지면 본 정책의 §7 만 먼저 채택해도 된다.
- **routing matrix drift** — selector 가 새 fallback 정책 (예: `fallback_security`) 을 추가하면 본 정책 §6.1 표가 즉시 stale. 다음 액션: §6.1 과 `role_selection.py` 사이의 정합성을 자동 점검하는 단위 테스트.
- **tech-lead degrade SPOF** — 모든 외부 surface 의 단일 author. 본 정책은 정확성 > 가용성 으로 수용했지만, 운영 중 tech-lead degrade 빈도 통계가 쌓이면 fallback 정책 (예: 6 역할 raw take 를 tech-lead 가 down 일 때 한해 외부에 노출하는 read-only mode) 을 다시 검토.
- **다음 작업**: 본 정책 §11 의 next-action 표 6 행. 우선 `run_supervisor_watch_loop` 의 self-improvement wiring 을 닫는 작업부터.
