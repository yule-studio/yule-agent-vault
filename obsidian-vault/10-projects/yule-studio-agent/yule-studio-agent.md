# yule-studio-agent — vault project root

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-12 | engineering-agent/tech-lead | 날짜 제거 + dash 컨벤션 + 폴더 README hub 구조 |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 vault README (날짜 prefix 컨벤션) |

## 이 vault 가 묶는 것

운영자 + 자동화 에이전트가 함께 쓰는 **yule-studio-agent 프로젝트의 모든
결정 / 리서치 / 작업 로그 / 운영 지식 노트** 가 모이는 root. 다음 가지를
타고 들어가면 모든 노트에 도달한다.

## 가지 (폴더 README)

- [[_moc/_index|_moc/]] — Map of Content (주제별 hub: F15 / manifest /
  Discord / Hermes / 등)
- [[decisions/README|decisions/]] — 결정 노트 (`decision-*.md`)
- [[research/README|research/]] — 리서치 노트 (`research-*.md`)
- [[task-logs/README|task-logs/]] — 작업 로그 (`task-log-*.md`)
- [[knowledge/README|knowledge/]] — 운영 지식 (`knowledge-*.md` +
  `knowledge/plugins/`)

## 파일명 컨벤션 (v.2.0.0)

```
<kind>-<topic-kebab>[-issue-<n>].md
```

- `<kind>` — `decision` / `research` / `task-log` / `knowledge` /
  `meeting` / `work-report`
- `<topic-kebab>` — kebab-case, 영문 소문자/숫자/하이픈만
- `-issue-<n>` — 선택. 관련 GitHub issue 번호

**날짜는 파일명에 넣지 않는다.** 날짜는 frontmatter `created_at` +
본문 첫 문서 버전 표 에서만 관리. 파일명이 버전과 함께 바뀌면
[[link]] 가 깨지므로 안정적 식별자가 필요.

예 (표준):

- `decision-product-vs-marketing-cpo-cmo-separation-issue-126.md`
- `task-log-tech-lead-runtime-loop-issue-73.md`
- `research-engineering-knowledge-surface-strengthening.md`

비표준 (금지):

- `2026-05-08_decision_tech-lead-single-write-subject.md`
  → 날짜 prefix 금지 (mistake_ledger `obsidian.filename.date-prefix`)
- `2026-05-08_59-hermes-tech-lead.md`
  → kind prefix 누락
- `hermes-yule-integration-decisions.md`
  → kind 단수 + 앞쪽 위치 (`decision-hermes-yule-integration`)

검증: `validate_filename` (`src/yule_orchestrator/agents/obsidian/
filename_convention.py`) 가 hard rail. CI 회귀 테스트
(`tests/engineering/test_obsidian_convention_governance.py`) 가 영구
가드.

## 본문 첫 줄 표준 (v.2.0.0)

모든 노트는 frontmatter 다음에 다음 표를 둔다 (이 README 처럼):

```markdown
# <한 줄 제목>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <부서>/<역할> | 최초 ... |
```

이 표가 파일명에서 빠진 날짜·버전 정보를 보존한다. 후속 개정마다 행을
추가 (위쪽이 최신).

## frontmatter 표준

```yaml
---
title: "한 줄 제목"
kind: decision | research | task-log | knowledge | meeting | work-report
project: yule-studio-agent
agent: <부서>/<역할>
status: decided | draft | superseded | current | completed | historical
created_at: 2026-MM-DDTHH:MM:SS+09:00
session_id: <짧은 식별자>  # optional
issue_number: <NN>          # optional
related:
  - ../<폴더>/<관련-노트>.md
tags:
  - <topic-tag>
---
```

## 자동화 에이전트 작성 규칙

1. 본 컨벤션 (파일명 + 본문 표 + frontmatter) 준수
2. PasteGuard 통과한 페이로드만 작성
3. 해당 폴더 README 에 [[link]] 추가 (가지에 연결)
4. 같은 주제의 다른 단계 노트 (decision/research/task-log) 와
   frontmatter `related` 로 cross-link
5. 새 주제면 `_moc/` 안에 hub 노트도 함께 작성

## 관련 repo 문서

- `policies/runtime/vault/naming-convention.md` — 본 컨벤션의 repo 사본
- `policies/runtime/agents/engineering-agent/issue-pr-conventions.md` §4.1 — hard rail
