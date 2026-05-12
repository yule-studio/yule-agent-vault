# obsidian-vault — 인덱스

> Obsidian 이 직접 여는 vault. PARA-like 10 폴더 구조 (00-90).
> 모든 노트는 본 README 의 가지 어딘가에 속한다.

**[[../index|↑ yule-agent-vault 진입점]]**

---

## 가지 (10 폴더)

| # | 폴더 | 진입점 | 한 줄 책임 |
| --- | --- | --- | --- |
| 00 | inbox | [[00-inbox/inbox]] | 분류 전 임시 (links / quick-notes / unsorted) |
| 10 | projects | [[10-projects/projects]] | 프로젝트별 (yule-studio-agent / homelab / spring-sandbox / yule-smoke-test) |
| 20 | areas | [[20-areas/areas]] | 지속 학습 주제 (ai-agent / backend / devops / frontend / career / computer-science) |
| 30 | resources | [[30-resources/resources]] | 외부 자료 요약 (articles / books / lectures / official-docs / papers / examples) |
| 40 | patterns | [[40-patterns/patterns]] | 재사용 설계 패턴 (architecture / api / design / deployment / error-handling / refactoring) |
| 50 | snippets | [[50-snippets/snippets]] | 코드 조각 (bash / docker / java / python / spring / sql / typescript / github-actions) |
| 60 | troubleshooting | [[60-troubleshooting/troubleshooting]] | 에러 해결 (ai-agent / backend / database / devops / frontend / network) |
| 70 | daily | [[70-daily/daily]] | 일일 작업 기록 (연도별) |
| 80 | templates | [[80-templates/templates]] | Obsidian 노트 템플릿 |
| 90 | archive | [[90-archive/archive]] | 보관 (deprecated-notes / migrated / old-projects) |

## 어디에 노트를 두나? (라우팅 표)

| 노트 종류 | 위치 |
| --- | --- |
| 분류 미정 | `00-inbox/quick-notes/` 또는 `00-inbox/unsorted/` |
| 프로젝트 결정 / 리서치 / 작업 로그 | `10-projects/<project>/{decisions,research,task-logs}/` |
| 개념 정리 ("Spring JPA 란?") | `20-areas/<area>/` |
| 외부 자료 요약 ("Effective Java 5장") | `30-resources/{books,articles,...}/` |
| 재사용 패턴 ("Saga pattern") | `40-patterns/<category>/` |
| 코드 한 조각 (≤50 줄) | `50-snippets/<lang-or-tool>/` |
| 에러 해결 기록 | `60-troubleshooting/<area>/` |
| 오늘 한 일 | `70-daily/<year>/<YYYY-MM-DD>.md` |
| 템플릿 / 양식 | `80-templates/` |
| 더 이상 안 쓰는 노트 | `90-archive/<category>/` |

자동화 에이전트의 export 라우팅: [[../CLAUDE|../CLAUDE.md]]

## 파일명 컨벤션 (v.2.0.0)

```
<kind>-<topic-kebab>[-issue-<n>].md
```

- `<kind>` ∈ `decision` / `research` / `task-log` / `knowledge` /
  `meeting` / `work-report` / `concept` / `troubleshooting` /
  `snippet` / `pattern`
- `<topic-kebab>` — 소문자 / 숫자 / hyphen
- `-issue-<n>` — 선택, GitHub issue 번호
- **날짜는 파일명에 넣지 않는다** — frontmatter `created_at` + 본문
  문서 버전 표 에서만 관리

검증: yule-studio-agent repo 의 `filename_convention.validate_filename`
+ `tests/engineering/test_obsidian_convention_governance.py`

## 본문 첫 줄 표준

```markdown
# <한 줄 제목>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <부서>/<역할> | 최초 ... |
```

후속 개정마다 행 추가 (위쪽이 최신). 파일 rename 없이 새 버전 행만
추가하므로 [[link]] 가 깨지지 않음.

## frontmatter 표준

```yaml
---
title: "한 줄 제목"
kind: decision | research | task-log | knowledge | concept | troubleshooting | snippet | pattern
project: <프로젝트명 if 10-projects>
agent: <부서>/<역할>
status: decided | draft | superseded | current | completed | historical
created_at: 2026-MM-DDTHH:MM:SS+09:00
related:
  - ../<폴더>/<관련-노트>.md
tags:
  - <topic-tag>
---
```

## 운영 규칙 한 줄

1. 어디에 둘지 모르면 `00-inbox/unsorted/` → 나중에 분류.
2. 같은 노트를 두 폴더에 두지 않는다 (cross-link 로 연결).
3. 단순 복붙이 아닌 **요약 + 내 해석** 을 항상 포함.
4. 폴더 README 의 "현재 노트" 인덱스 갱신.
5. 컨벤션 위반 시 mistake_ledger 에 등록되어 CI 가드 fail.

## 관련

- [[../index|↑ yule-agent-vault README]]
- [[../CLAUDE|운영 규칙 (CLAUDE.md)]]
- yule-studio-agent repo: `policies/runtime/vault/naming-convention.md`
