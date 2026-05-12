---
title: "engineering-knowledge surface 강화 — share boundary + evidence 라벨 검토"
kind: research
session_id: company-runtime-knowledge-surface
project: yule-studio-agent
created_at: 2026-05-10T00:00:00+09:00
agent: engineering-agent/tech-lead
related:
  - ../decisions/decision-engineering-knowledge-share-boundary.md
  - ../task-logs/task-log-company-runtime-knowledge-surface.md
tags:
  - research
  - engineering-intelligence
  - knowledge-surface
  - share-scope
  - retrieval
status: historical
---

# 목표

structured knowledge 결과(`EngineeringKnowledgeItem`)가 Obsidian 에 land 한 뒤, 외부 surface(Discord 일별 digest, PR body, 합성 응답)에 어떻게 인용되어야 하는지 명확히 한다. 또 retrieval 점수의 트레이스(`KnowledgeMatch.signals`)를 사람이 읽을 수 있는 형태로 surface 에 남기는 방법을 정한다.

# 배경

commit `a13de21` 까지의 Round 4 흐름:

1. background scheduler 가 source seed 를 돌며 `EngineeringKnowledgeItem` 을 만든다.
2. quality gate (`obsidian.evaluate_quality_gate`) 가 16 항목을 검증.
3. renderer 가 frontmatter + 13 섹션 markdown 으로 변환, vault 에 L1 자동 저장.
4. request 시점에 `KnowledgeRetriever` 가 `KnowledgeRecord` 후보를 score signal 과 함께 정렬, 상위 N 개를 `ContextPack.relevant_knowledge` 에 끼워 넣음.
5. discussion 합성기 / Discord digest / PR body footer 가 `relevant_knowledge` 를 surface 에 노출.

그러나 5번에서 다음 두 gap 이 드러난다:

- **share boundary 부재**: 한 자료가 PUBLIC 인지(공식 docs), TEAM_INTERNAL 인지(사내 ADR), RESTRICTED 인지(보안 incident) 가 contract 에 박혀 있지 않다. 외부 surface 코드가 자료의 source_kind / source_url 을 보고 추측해야 하는데, 코드 위치가 흩어져 있다.
- **retrieval evidence 라벨 부재**: `signals` 가 raw 토큰(`role_primary_match` / `axis_overlap:...`)이라 운영자가 surface 에서 보기 어렵다. 합성 응답이 "이 자료를 왜 골랐는가" 를 자동으로 설명하기에 단계가 모자란다.

# 검토한 옵션

## share boundary 를 어디에 두나

### 옵션 A — collector 단계의 source_entry 에 박기

`SourceEntry.content_policy: str` 이 이미 있고 "link + summary only" 같은 자유 텍스트가 들어 있다. 여기에 share_scope 를 박을 수도 있다.

장점: source 단위로 한 번 결정하면 모든 자료가 자동 상속.

단점: source_entry 는 transport metadata(어떻게 fetch 하나)와 트러스트 가중치 위주이고, 자료별 share_scope 는 같은 source 라도 달라질 수 있다(공개 release note 와 사내 release note 가 같은 source 도구에서 fetch). 단계가 collector 안쪽이라 한 번 결정되면 surface 에서 override 가 불가능하다는 것이 결정적이다.

### 옵션 B — `EngineeringKnowledgeItem` 자체에 박기 (선택)

`item.share_scope` 필드로 박는다. 자료 단위 결정이라 운영자 검토 단계에서 override 가능. quality gate 가 share_scope 와 reason 의 일관성(restricted 면 reason 강제)을 본다.

장점: 자료 단위 정확도, surface 코드가 단일 enum 만 보면 됨, 후일 enum 확장도 자료 단위 마이그레이션만 하면 됨.

단점: collector 가 한 번 더 결정하는 단계가 필요. → 본 PR scope 에선 default `public` 으로 두고, 운영자가 명시적으로 override 하는 흐름. 후속 PR 에서 source_kind → share_scope 자동 매핑을 더하면 된다.

→ **옵션 B 선택**.

## restricted 본문은 어디에서 자르나

### 옵션 A — collector 가 본문을 안 채운 채 vault 에 저장

자료가 vault 에 land 했을 때 이미 본문 없음. 후일 share_scope 가 PUBLIC 으로 승격되면 자료를 다시 fetch 해야 함.

### 옵션 B — renderer 가 markdown 출력 시 마스킹 (선택)

dataclass 자체는 본문을 갖고 있고, vault markdown body 는 placeholder. 후일 승격 시 dataclass 그대로 다시 render → 본문 출력.

→ **옵션 B 선택**. 자료의 information 자체를 잃지 않는 것이 자산화 정책에 맞다.

## evidence 라벨 변환을 어디서

### 옵션 A — surface 코드 (synthesizer / discord_summary) 가 각자 변환

label 매핑이 surface 마다 다를 위험.

### 옵션 B — `KnowledgeMatch.evidence_labels()` 한 곳에서 변환 (선택)

surface 는 `match.evidence_labels()` 만 호출. 새 signal 추가 시 `_KNOWN_SIGNAL_LABELS` 한 곳만 갱신.

→ **옵션 B 선택**.

# 외부 자료

share boundary 의 mental model 은 다음 자료를 참고했다:

- [Mozilla "Sensitive Information" guidance](https://wiki.mozilla.org/Security/Guidelines/Web_Security#Sensitive_Information_Handling) — 본문/링크/메타 별로 다른 ACL 을 권장.
- [GOV.UK Service Manual: Sharing data safely](https://www.gov.uk/service-manual/technology/sharing-data-safely) — 자료의 노출 단계를 분리할 때 enum/tier 가 자유 텍스트보다 운영 risk 가 낮다.

(외부 자료는 share_scope=public 으로, 본 research note 의 참고 자료 섹션에 그대로 인용.)

# 권고

본 PR 가 land 한 contract 를 지킨다는 것은 다음 4 줄로 요약된다:

1. 모든 외부 surface 코드는 `obsidian.shareable_external_payload(item)` 만 호출한다.
2. 합성 응답의 evidence 블록은 `ContextPack.format_knowledge_evidence_block()` 만 호출한다.
3. discord summary 는 `discord_summary.render_daily_role_summary` 그대로. share_scope 분기는 안에 박혀 있다.
4. 새 share_scope 가 필요해지면 `KnowledgeShareScope` enum 확장 + 본 결정 노트 갱신.

# 후속 검토 후보

- source_kind 별 default share_scope 매핑(현재는 모두 default PUBLIC).
- 합성 응답이 evidence 블록을 어디에 끼워 넣어야 가장 자연스러운지 (footer / inline / 별도 thread reply).
- vault index 가 share_scope 필터로 자료를 빠르게 추리는 API.
