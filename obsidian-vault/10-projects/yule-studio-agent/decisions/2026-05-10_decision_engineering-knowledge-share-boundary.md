---
title: "engineering-knowledge note share_scope boundary 결정"
kind: decision
session_id: company-runtime-knowledge-surface
project: yule-studio-agent
created_at: 2026-05-10T00:00:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - ../research/2026-05-10_research_engineering-knowledge-surface-strengthening.md
  - ../task-logs/2026-05-10_task-log_company-runtime-knowledge-surface.md
tags:
  - decision
  - engineering-intelligence
  - knowledge-surface
  - share-scope
  - boundary
---

# 목표

수집된 자료(`EngineeringKnowledgeItem`)가 외부 surface(Discord 일별 digest, PR body, 합성 응답)에 어느 범위까지 인용 가능한지를 contract 수준에서 박는다. "운영자가 합의했지만 코드는 모르는" 상태를 끝낸다.

# 결정

| ID | 항목 | 결정 |
| --- | --- | --- |
| D-KS-1 | share_scope 표현 | enum (`public` / `team_internal` / `restricted`) |
| D-KS-2 | enum 위치 | `engineering_intelligence.models.KnowledgeShareScope` |
| D-KS-3 | 기본값 | `public` (collector 가 명시 안 하면 PUBLIC) |
| D-KS-4 | restricted 강제 reason | `quality_gate.missing:share_scope_reason_for_restricted` 로 막음 |
| D-KS-5 | restricted 본문 마스킹 위치 | renderer (14 섹션 중 학습 본문 10 섹션 placeholder 대체) |
| D-KS-6 | 외부 surface 인용 경로 | `obsidian.shareable_external_payload(item)` 단일 진입점 |
| D-KS-7 | 합성 응답 evidence 블록 | `discussion.context_pack.ContextPack.format_knowledge_evidence_block` |
| D-KS-8 | retrieval 근거 라벨 위치 | `engineering_intelligence.retrieval.label_for_signal` (한국어 라벨) |

# 각 결정의 의미

## D-KS-1 / D-KS-2 enum 으로 박는다

string 자유필드로 두면 collector / vault index / surface 가 각자 다른 표기를 쓸 위험이 있다. enum 으로 박으면 typo 시 ValueError 가 즉시 발생, 새 scope 추가 시 한 군데에서 검색해 영향 범위를 본다.

## D-KS-3 기본값 PUBLIC

기존에 vault 에 들어간 모든 자료는 공식 docs / release notes / blog 류였고, 운영자 검토 결과 PUBLIC 이 맞다. 새 자료 카테고리(사내 ADR, security incident)가 PUBLIC 으로 잘못 들어가는 것을 막으려면 collector 단계에서 source_kind 별 기본 매핑을 추가하는 것이 후속 작업으로 합당하다. 본 PR 단계에서는 운영자 명시적 override 만 인정한다.

## D-KS-4 restricted 는 reason 강제

`restricted` 자료는 본문 인용을 차단하는 강한 신호다. 이유 없이 land 하면 후일 운영자가 왜 이 자료가 vault 에서만 보이는지 추적할 수 없게 된다. quality gate 에서 `missing:share_scope_reason_for_restricted` 로 막아 reason 필드를 강제한다.

## D-KS-5 restricted 본문 마스킹은 renderer 에서

본문 마스킹을 collector 단계에서 하면 vault 가 자료를 "잃어버린다". renderer 단계에서 하면 dataclass 자체는 본문을 들고 있지만, vault note (markdown body) 와 외부 surface 양쪽 모두 본문을 보지 못한다. 즉 후일 share_scope 를 PUBLIC 으로 승격해도 자료를 다시 fetch 할 필요가 없다.

마스킹 대상 = 학습 본문 10 섹션. 유지 = 문서 개요 / RAG-CAG 메타 / 공유 가능 범위 / 참고 자료. 자료가 vault 에 어떤 키워드로 추적되는지는 surface 에 남고, 본문/실습/공통 실수 같은 운영 본문만 가린다.

## D-KS-6 외부 surface 는 `shareable_external_payload` 만 사용

raw `EngineeringKnowledgeItem.to_payload()` 는 vault 전용. 외부 surface 코드가 이 dict 를 그대로 들고 가면 share_scope 를 무시한 본문 노출이 가능해진다. 단일 진입점 헬퍼를 두고, payload shape 자체가 share_scope 별로 다르도록 박는다 (`public` 만 summary 포함, `team_internal` 은 summary 차단, `restricted` 는 marker + reason 만).

## D-KS-7 evidence 블록은 ContextPack 에 박는다

discussion 합성기가 직접 `relevant_knowledge` 를 dump 하면 공통 share_scope 처리가 누락되기 쉽다. `ContextPack.format_knowledge_evidence_block()` 단일 메서드로 박아 두면 호출자(synthesizer / decision-note builder / PR body footer) 모두 동일한 처리를 받는다.

## D-KS-8 한국어 evidence 라벨

raw signal 토큰(`role_primary_match`, `axis_overlap:api_schema_auth,...`, `topic_overlap:2`)은 운영자가 읽기 어렵다. `label_for_signal` 단일 매핑으로 한국어 라벨로 변환해서 합성 응답 / Obsidian decision note 어디서나 동일하게 노출.

# 비결정 (의도적)

- collector 단계에서 source_kind → share_scope 자동 매핑은 본 PR 에 포함 안 함. 운영자가 source seed 를 추가할 때 share_scope 를 명시하는 흐름을 한 번 거쳐, 어떤 디폴트 매핑이 적절한지 데이터를 모은 뒤 후속 PR 에서 결정.
- `confidential` (4번째 단계) 도입 안 함. 현재까지 수집되는 자료 패턴에서 `restricted` 이상의 분류가 필요한 케이스가 없었다. 필요해지면 enum 에 추가하면 된다.

# 변경 영향

- 기존 `EngineeringKnowledgeItem` 인스턴스: 모두 `share_scope=PUBLIC` default 로 처리, 운영 surface 동작 동일.
- write request metadata: `engineering_intelligence.share_scope` + `shareable_external_payload` 가 추가됨. 기존 메타 키는 유지.
- ContextPack `as_dict()`: `relevant_knowledge[*]` 에 `share_scope` / `share_scope_reason` / `evidence_labels` 키 추가.
- 회귀: 3,203 테스트 통과.
