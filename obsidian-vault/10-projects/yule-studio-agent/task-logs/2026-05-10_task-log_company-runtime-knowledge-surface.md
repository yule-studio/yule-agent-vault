---
title: "engineering_intelligence note/report surface 강화 — 작업 로그"
kind: task-log
issue: 73
parent_issue: 20
session_id: company-runtime-knowledge-surface
project: yule-studio-agent
created_at: 2026-05-10T00:00:00+09:00
status: in-progress
branch: feature/company-runtime-knowledge-surface
worktree: /Users/masterway/local-dev/yule-studio-agent-worktrees/company-runtime-knowledge-surface
mode: tech-lead-mediated (engineering_intelligence note/report formatting)
related:
  - ../decisions/2026-05-10_decision_engineering-knowledge-share-boundary.md
  - ../research/2026-05-10_research_engineering-knowledge-surface-strengthening.md
tags:
  - task-log
  - engineering-intelligence
  - knowledge-surface
  - share-scope
---

# 목표

`feature/company-runtime-knowledge-loop` 위에서 structured knowledge 결과가 Obsidian / human-readable surface 에 더 잘 남도록 강화한다. 수집된 자료가 단순히 vault 어딘가에 쌓이는 것이 아니라, 운영 메모리 자산으로 보이고 재사용 가능해야 한다.

세부 4 항목:

1. structured knowledge output contract 를 note/report 수준에서 명확화
2. 공유 가능/불가 정보 경계와 note section 규칙 강화
3. retrieval 결과가 discussion 에 어떤 근거로 쓰였는지 surface 에 남길 수 있게 포맷 보강
4. 테스트 추가

# 현재 Yule 기준선 (commit a13de21 시점)

- `engineering_intelligence/models.py` 의 `EngineeringKnowledgeItem` 은 contract id `engineering-knowledge/v0` 로 박혀 있고, frontmatter + 13 섹션 + quality gate 16 항목을 통과해야 vault 에 land 한다.
- `engineering_intelligence/retrieval.py` 의 `KnowledgeRetriever` 가 score signal (`role_primary_match`, `axis_overlap:...`, `topic_overlap:N`, `importance_*`, `fresh_*d`, `empty_body_penalty`)을 만들어 주고 있다.
- `discussion/context_pack.py` 의 `ContextPackBuilder` 가 `relevant_knowledge` 슬롯을 채우지만, 채워진 자료가 합성 응답에 어디까지 인용되어도 되는지 결정하는 share boundary 가 contract 에 박혀 있지 않다.
- 운영 surface (Discord digest, PR body, 합성 응답) 는 `to_payload()` 결과나 vault note 본문을 그대로 카피해서 노출할 수 있는 위험이 있다.

# 진행 단계 (실시간 갱신)

| 시점 | 단계 | 상세 |
| --- | --- | --- |
| 2026-05-10 kickoff | sub-task + 본 worktree | 사용자가 본 round 의 4 항목 정리. branch `feature/company-runtime-knowledge-surface`. |
| 2026-05-10 commit-1 | engineering_intelligence note/report contract 강화 + share boundary | `KnowledgeShareScope` enum + `share_scope`/`share_scope_reason` 필드, renderer 14 섹션 (신규: 공유 가능 범위), restricted 본문 마스킹, quality gate 가드, `shareable_external_payload` / `vault_only_metadata` 헬퍼, Discord summary share-scope 분기, retrieval `evidence_labels()` + `label_for_signal`. 테스트 22 건 추가. (commit f8a3dc6) |
| 2026-05-10 commit-2 | discussion ContextPack evidence surface + 정책/Obsidian 노트 | `EngineeringKnowledgeRef` 에 share_scope/evidence_labels 추가, `ContextPack.format_knowledge_evidence_block()` 신규, `ContextPackBuilder._select_relevant_knowledge` 가 retriever `with_signals` 우선 호출, `_coerce_knowledge_ref` 가 `KnowledgeMatch` 도 흡수. policies/runtime/agents/engineering-agent/research-collector.md §13.3 + research-profiles.md 부록 C 추가. 본 task-log/decision/research 노트 land. | 본 commit |

# 변경 / 산출물

## 코드

- `src/yule_orchestrator/agents/engineering_intelligence/models.py` — `KnowledgeShareScope` 신규.
- `src/yule_orchestrator/agents/engineering_intelligence/renderer.py` — 14 섹션, share_scope 본문 분기.
- `src/yule_orchestrator/agents/engineering_intelligence/obsidian.py` — quality gate 가드 + `shareable_external_payload` / `vault_only_metadata`.
- `src/yule_orchestrator/agents/engineering_intelligence/discord_summary.py` — share scope 별 라인 포맷.
- `src/yule_orchestrator/agents/engineering_intelligence/retrieval.py` — `evidence_labels()` + `label_for_signal`.
- `src/yule_orchestrator/agents/discussion/context_pack.py` — `EngineeringKnowledgeRef` 확장, `format_knowledge_evidence_block`.

## 정책

- `policies/runtime/agents/engineering-agent/research-collector.md` §13.3 신규 — share_scope 외부 surface 인용 boundary.
- `policies/runtime/agents/engineering-agent/research-profiles.md` 부록 C 신규 — share_scope 별 surface 동작 + evidence label 매핑.

## 테스트

- `tests/engineering_intelligence/test_share_scope.py` (16 신규) — frontmatter / 14 섹션 / RESTRICTED 본문 마스킹 / quality gate / shareable payload / vault-only metadata / Discord summary 분기.
- `tests/engineering_intelligence/test_retrieval.py` (6 신규) — evidence label 변환 + share_scope mapping coercion.
- `tests/engineering_intelligence/test_renderer.py` — 14 섹션 가정에 맞춰 TOC 길이 검증을 동적으로.
- `tests/engineering/test_discussion_context_pack.py` (7 신규) — evidence block 출력 share-scope 별 분기 + KnowledgeMatch unwrap path.

# 결정/설계

[[2026-05-10_decision_engineering-knowledge-share-boundary]] 참고. 핵심: share_scope 는 enum 으로 contract 에 박는다. 외부 surface 가 직접 raw `to_payload()` 를 들고 가지 않고 `shareable_external_payload(item)` 만 사용하게 만들어, 운영자 합의가 코드 보호로 승격된다.

# 검증

- `python3 -m unittest discover -s tests -t .` 3,203 케이스 통과 (skipped=5).

# 후속 작업 (별도 PR)

1. discussion `synthesizer.py` 가 `pack.format_knowledge_evidence_block()` 결과를 합성 응답 footer 에 삽입하도록 wiring (본 worktree write scope 외).
2. `obsidian_writer_worker` 가 `engineering-knowledge` write request 의 `shareable_external_payload` 메타를 보고 share_scope 별 별도 sidecar 를 저장하도록 옵션 추가.
3. discord 일별 digest 가 동일 share_scope 의 자료를 한 번에 묶어 표시하는 그루핑 옵션.
