---
title: "Vault README → 폴더명.md 일괄 rename + 최상단부터 가지 연결"
kind: task-log
session_id: vault-readme-rename-2026-05-12
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: completed
created_at: 2026-05-12T17:00:00+09:00
issue_number: 126
related:
  - task-log-vault-sweep-graph-consolidation.md
  - ../_moc/_moc.md
  - ../yule-studio-agent.md
tags:
  - task-log
  - vault
  - readme
  - rename
  - graph
  - issue-126
---

# Vault README → 폴더명.md rename

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

## 목표

사용자 요청 2 건 동시 처리:

1. **모든 폴더 README 가 README.md 라 그래프에서 구분이 안 된다** —
   18 개 노드가 다 "README" 라벨로 나타남. 최상단 `yule-agent-vault/
   README.md` 만 README 로 두고, 나머지는 폴더명에서 숫자 prefix 만
   뺀 이름으로 rename.
2. **최상단 README 부터 모든 가지 (00-09 ~ 90-archive) 가 연결되어야**
   한다. 최상단 → obsidian-vault/index → 10 폴더 인덱스 → 각 폴더의
   하위 영역.

## 진행

### 1. 18 README rename

| old | new |
| --- | --- |
| `obsidian-vault/README.md` | `obsidian-vault/index.md` |
| `00-inbox/README.md` | `00-inbox/inbox.md` |
| `10-projects/README.md` | `10-projects/projects.md` |
| `20-areas/README.md` | `20-areas/areas.md` |
| `30-resources/README.md` | `30-resources/resources.md` |
| `40-patterns/README.md` | `40-patterns/patterns.md` |
| `50-snippets/README.md` | `50-snippets/snippets.md` |
| `60-troubleshooting/README.md` | `60-troubleshooting/troubleshooting.md` |
| `70-daily/README.md` | `70-daily/daily.md` |
| `80-templates/README.md` | `80-templates/templates.md` |
| `90-archive/README.md` | `90-archive/archive.md` |
| `yule-studio-agent/README.md` | `yule-studio-agent/yule-studio-agent.md` |
| `decisions/README.md` | `decisions/decisions.md` |
| `research/README.md` | `research/research.md` |
| `task-logs/README.md` | `task-logs/task-logs.md` |
| `knowledge/README.md` | `knowledge/knowledge.md` |
| `knowledge/plugins/README.md` | `knowledge/plugins/plugins.md` |
| `_moc/README.md` | `_moc/_moc.md` |

최상단 `yule-agent-vault/README.md` 만 README 로 유지 (오픈소스
컨벤션).

### 2. 모든 cross-link 갱신

- markdown link `[label](path/to/README.md)` → 새 파일명
- Obsidian wiki link `[[..]/README]]` → `[[../<folder>]]`
- frontmatter `related:` 의 `_index.md` → `_moc.md`
- 총 **26 파일 patched**

### 3. 가지 연결 (최상단부터)

```
yule-agent-vault/README.md  ← 진입점 (오픈소스 README 스타일)
└── obsidian-vault/index.md  ← vault 인덱스 (10 폴더 표)
    ├── 00-inbox/inbox.md
    ├── 10-projects/projects.md
    │   └── yule-studio-agent/yule-studio-agent.md
    │       ├── _moc/_moc.md (+ _moc/_index.md)
    │       ├── decisions/decisions.md
    │       ├── research/research.md
    │       ├── task-logs/task-logs.md
    │       └── knowledge/knowledge.md
    │           └── plugins/plugins.md
    ├── 20-areas/areas.md
    ├── 30-resources/resources.md
    ├── 40-patterns/patterns.md
    ├── 50-snippets/snippets.md
    ├── 60-troubleshooting/troubleshooting.md
    ├── 70-daily/daily.md
    ├── 80-templates/templates.md
    └── 90-archive/archive.md
```

## 결정 요약

### 왜 폴더명을 파일명으로 썼나

- **그래프 가독성** — 모든 노드가 "README" 가 아니라 `inbox` /
  `projects` / `areas` / ... 로 표시되어 한 눈에 어떤 폴더 인덱스인지
  보임.
- **링크 텍스트 명료성** — `[[../decisions/decisions]]` 형식이
  `[[../decisions/README]]` 보다 의도 명확.
- **숫자 prefix 제거 이유** — 파일명에 `00-` `10-` 같은 prefix 가 있으면
  Obsidian 검색 / autocomplete 에서 사용자가 입력해야 할 토큰이
  늘어남. 폴더명은 정렬 위해 숫자 유지하되 안의 인덱스 파일은
  의미 토큰만.

### 왜 최상단만 README 로 두었나

- GitHub / 오픈소스 컨벤션 — 레포 루트에 `README.md` 가 있어야 외부에서
  자동 표시.
- 내부 vault 는 Obsidian 그래프 / 검색 친화 형식이 우선.

### `_moc/` 의 두 파일 (`_moc.md` + `_index.md`) 의 역할 분리

- `_moc/_moc.md` — 폴더 README 역할 (인덱스 + 운영 규칙). 다른 폴더의
  `<folder>.md` 와 같은 패턴.
- `_moc/_index.md` — 그래프 hub 역할 (주제별 hub 의 인덱스). hub 간
  관계 + 새 hub 추가 체크리스트.
- 두 파일은 의도적으로 분리 유지 — 폴더 메타 (README 역할) vs 그래프
  hub 인덱스.

## 비결정 / 비실행

- **vault root 의 0-line stub 파일 2 개**
  (`2026-05-08_issue-69-decision-engineering-agent-authoring-policy.md`,
  `2026-05-08_issue-69-research-engineering-agent-governance-synthesis.md`)
  — 안전장치로 삭제 거부. 사용자가 직접 삭제 필요.
- **기존 노트 본문의 broken [[link]] 문장** — 본문 수정이 안전장치로
  막혀 frontmatter / cross-link 만 일괄 자동 갱신됨.

## 다음 행동

1. vault git commit + push (`yule-agent-vault` repo, origin
   `yule-studio/yule-agent-vault`)
2. Obsidian 그래프 reload 후 가독성 확인
3. F15 잔여 commit (yule-studio-agent repo)

## 회귀

repo 변경 없음 (본 작업은 vault-only). yule-studio-agent repo 의
`tests/engineering/test_obsidian_convention_governance.py` 가 vault
파일명 컨벤션 검증 — 본 rename 후에도 `<kind>-<topic>.md` 파일들은
모두 통과 (folder README 는 `<folder>.md` 라 컨벤션 검증 대상 외).
