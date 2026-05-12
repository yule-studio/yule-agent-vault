---
title: "결정 — engineering-agent 의 외부 write 주체는 tech-lead 1 명으로 단일화"
kind: decision
project: yule-studio-agent
issue: "#48"
branch: feature/agent-company-harness-48
decided_at: "2026-05-08"
author: engineering-agent/tech-lead
status: decided
contract: decision/v0
agent: engineering-agent/tech-lead
created_at: 2026-05-08T00:00:00+09:00
---

# 결정 — engineering-agent 의 외부 write 주체는 tech-lead 1 명으로 단일화

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-08 | engineering-agent/tech-lead | 최초 결정 — issue #48 |

## 목표

Yule engineering-agent 가 GitHub / Discord / Obsidian 등 외부 surface 로 발화 / 커밋 / 노트를 남길 때, **author / commenter / committer / PR 작성자 / vault note author** 의 책임 주체를 정확히 1 명으로 못박는다.

## 결정 한 줄 요약

> **engineering-agent 의 모든 외부 write 주체는 `engineering-agent/tech-lead` 1 명이다. backend / frontend / devops / qa / ai-engineer / product-designer 는 분석 관점만 제공하며, 자기 take 를 직접 외부 surface 로 발화하지 않는다.**

## 현재 Yule 기준선

- **lifecycle-mvp.md §3** 13 단계 lifecycle 에서 `synthesis` 와 `final_report` 는 tech-lead 가 책임. 단, 외부 회신을 누구의 이름으로 내보내는지의 정책 표면은 명시적으로 한 곳에 모여 있지 않았다.
- **role-profiles.md §"TechLeadAggregator 정책"** 이 6 역할 take 를 종합하는 helper 를 정의하지만, 종합 결과를 외부에 노출할 때의 author tone 은 implicit 였다.
- **agents/github_workos/triage.py** `senior_triage` 가 primary_role / support_roles / excluded_roles 를 한 본문 안에 묶어 senior engineer tone 으로 출력 — 본 결정의 자연스러운 선례.
- **agents/coding/authorization.py** 가 executor 추천 / reviewer 분리 / 사용자 승인 게이트를 운영 중 — 본 결정의 producer-reviewer 측 근거.
- **GitHub commit author** 는 별개로 `agents/github_workos/identity.py:COMMIT_AUTHOR_POLICY_OWNER_AS_AUTHOR` ("owner-as-author") 로 정의 — commit author 는 사람, committer 만 봇.

## 참고한 외부 레퍼런스

- 이슈 본문에 명시된 6 가지 팀 패턴 — 특히 Producer-Reviewer + Hierarchical Delegation 이 본 결정의 근거.
- Harness 의 "회사형 팀" 비유 — 외부 의사 결정이 흩어지지 않게 한 사람 (tech-lead) 이 외부 발화 주체로 등장.
- Yule 의 GitHub WorkOS senior_triage 의 single-author 접근 — 이미 운영 중인 선례.

## 도입한 부분

1. **모든 외부 surface 의 author = tech-lead 1 명**. 본 결정 한 줄 요약 그대로.
   - GitHub: issue comment, PR body, PR comment, draft → ready 전환, label 변경의 commenter / requester.
   - Discord: gateway 의 외부 회신, `#봇-상태` post.
   - Obsidian vault: research / decision / task-log / report / knowledge / engineering-knowledge note 의 author 필드.
   - commit: `committer` 자리. **author** 자리는 별개 정책 (`COMMIT_AUTHOR_POLICY_OWNER_AS_AUTHOR`) 이 사람을 박는다.
2. **6 역할은 분석 관점만**. 자기 take 를 외부에 직접 발화하지 않으며, take 는 `agent_ops_audit` + `role_take` 잡 결과로만 표현. tech-lead aggregator 가 충돌 / 의문문 / 사용자 결정 필요를 분리해 외부 회신에 단일 author tone 으로 합친다.
3. **Routing Matrix / Review Gate / Approval Gate** 정책 문서 §6 으로 못박아 운영자가 한 표에서 본다.
4. **L4 (force push / production deploy / merge / secret 변경)** 는 본 결정의 author 모델에서 **명시적으로 제외**. 사람이 직접 수행한다 — agent author 자체가 등장하지 않는 surface.

## 보류 / 비도입 부분

- **2 단 hierarchical delegation** (sub-team coordinator). 1 단 (engineering-agent → tech-lead → 6 역할) 가 안정화되기 전까지 보류.
- **6 역할의 author 권한 부분 부여** (예: backend 가 자기 stack 한정 vault note 직접 작성). 외부 surface 의 단일 tone 우선이라 본 단계에서는 거부.
- **commit author 자리에 봇 등장**. 운영 중인 owner-as-author 정책 그대로.
- **다른 부서 (platform / security / data-ai / product / design / marketing / operations)** 의 author 모델. team-structure.md 가 부서 단위 gateway 만 명시하고, 본 결정은 engineering-agent 1 부서로 한정. 다른 부서가 도입되면 같은 패턴을 부서별로 복제.

## 왜 실제 시니어 개발팀형 회사 구현에 필요한가

- 실제 회사에서는 외부 결정 발표가 한 사람의 이름으로 나간다. 6 명이 동시에 떠드는 outputs 는 내부 토론에서만 허용된다. 본 결정은 그 회사의 자연스러운 운영을 그대로 형식화한다.
- 외부 surface 의 author 가 분산되면 책임 추적이 무너진다 — issue comment 가 ai-engineer 이름으로, PR body 가 backend-engineer 이름으로, vault note 가 product-designer 이름으로 흩어지면, 어떤 issue 의 root-cause 가 어떤 author 의 판단인지 grep 단위로 답할 수 없다.
- selector / routing matrix 가 fallback 정책으로 불안정한 상태에서도 외부 author 가 일정해야 사용자가 "Yule 이 어떻게 결정했지?" 의 답을 한 곳 (tech-lead) 에서 받는다.

## 구현 / 설계 위치

- **정책 문서**: `policies/runtime/agents/engineering-agent/team-architecture-patterns.md` §5 (역할별 책임 — 실행 주체 vs 분석 관점 분리 기준).
- **연관 surface (이미 존재)**:
  - `tech_lead_aggregator.aggregate_role_outputs` — 6 역할 종합.
  - `agents/github_workos/triage.py:senior_triage` — single-author 본문 출력.
  - `agents/coding/authorization.py` — producer-reviewer 게이트.
  - `agents/job_queue/obsidian_writer_worker.py` — vault write subject 의 단일 surface.
- **연관 회귀 테스트**: `tests/engineering/test_team_architecture_patterns_doc.py` 가 본 결정의 핵심 문구 ("단일 write 주체", "write subject", protected branch / production deploy / force push 가 안전 rail) 를 정책 문서 안에서 anchor.
- **신규 코드 변경 0** — 본 결정은 author tone 의 정책 surface 이므로 code surface 는 손대지 않는다. 후속 PR 가 author 출력 surface 를 단일화할 때 본 결정 문서를 명시적 입력으로 받는다.

## 리스크 / 다음 액션

| 리스크 | 완화 / 다음 액션 |
| --- | --- |
| tech-lead degrade 시 외부 author 가 끊어진다 (SPOF) | 정확성 > 가용성 으로 수용. 운영 중 빈도 통계가 쌓이면 fallback 정책 재검토. |
| 6 역할 take 가 sanitised 되지 않은 채 audit log 로 새어 나가 외부 회신과 일관성 깨짐 | `forbidden_actions` gate 와 `agent_ops_audit` 의 sanitisation 정책을 정책 §6.2 review gate 로 못박음. |
| 외부 surface 가 늘어날 때마다 본 결정의 적용 범위를 다시 명시해야 함 | 정책 §5 표를 단일 source 로 못박고, 새 surface (예: Slack / Linear) 가 추가되면 표를 갱신. |
| commit author / committer 분리 정책과 본 결정의 충돌 가능성 | `COMMIT_AUTHOR_POLICY_OWNER_AS_AUTHOR` 그대로 유지. tech-lead 는 committer, 사람은 author — 본 결정이 의도적으로 commit author 자리는 건드리지 않음을 §"보류 / 비도입 부분" 에 명시. |

다음 작업:

1. 본 결정을 입력으로 후속 PR 가 외부 surface (Discord 회신 본문 / GitHub issue comment / vault note frontmatter) 의 author 필드를 단일화.
2. role-profiles.md / lifecycle-mvp.md 의 다음 개정에서 본 결정의 표 §5 를 cross-link.
3. role-profiles.md §"다른 role 로 확장할 때 재사용할 패턴" 7 단계를 적용해 frontend / devops / qa / ai / product-designer 의 v1 contract 도 점진 작성 — single author 의 분석 입력 품질을 끌어올리기 위함.
