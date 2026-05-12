---
title: "Vault sweep — 그래프 통합 + 폴더 README hub 구축"
kind: task-log
session_id: vault-sweep-2026-05-12
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: completed
created_at: 2026-05-12T15:30:00+09:00
issue_number: 126
related:
  - ../_moc/_moc.md
  - ../_moc/f15-corporate-structure.md
  - ../README.md
tags:
  - task-log
  - vault-sweep
  - obsidian
  - hub
  - issue-126
---

# Vault sweep — 그래프 통합

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 sweep 기록 |

## 목표

사용자 그래프 캡처 (2 회) 에서 노드들이 흩어져 있고 외톨이 / 회색
stub / broken backlink 가 다수 존재. "한 눈에 알아볼 수 있도록" +
"폴더링 명대로 연결" + "최상단 README 부터 가지 다 연결" 요청 충족.

## 진행 (시계열)

### 1. 컨벤션 정정

- 코드 `src/yule_orchestrator/agents/obsidian/filename_convention.py`
  에 이미 박혀 있던 컨벤션 (`<kind>-<topic>[-issue-<n>].md`, 날짜 금지,
  dash 분리자) 을 발견. 이전에 내가 underscore 형식으로 잘못 사용한
  것을 시정.
- `src/yule_orchestrator/agents/obsidian/export.py` + `knowledge_writer.py`
  의 basename 형식에서 날짜 제거 + dash 분리자로 변경
- 테스트 expectation 갱신 (4 파일), 회귀 1772 passed

### 2. 기존 18 노트 rename

- 16 노트 git mv + 3 노트 mv (untracked, 이번 세션 생성)
- 예: `2026-05-09_task-log_issue-73-tech-lead-runtime-loop.md` →
  `task-log-tech-lead-runtime-loop-issue-73.md`
- 모든 노트 본문의 cross-link 41 substitutions 갱신

### 3. 폴더 README hub 작성

- `README.md` (root) 갱신: v.2.0.0 — 컨벤션 v2 + 폴더 README 가지 +
  본문 표 표준 + frontmatter 표준
- `_moc/README.md` — MOC 인덱스 + 운영 규칙
- `decisions/README.md` — 5 결정 노트 인덱스 + 운영 규칙 4 개
- `research/README.md` — 4 리서치 노트 + 출처 규칙
- `task-logs/README.md` — 7 task-log + 작성 규칙 4 개
- `knowledge/README.md` — knowledge/plugins/ 11 노트 + 권위 source 규칙

### 4. _moc/ hub 6 개 작성

- `_moc/_index.md` (있음, 그래프 인덱스)
- `_moc/f15-corporate-structure.md` (있음)
- `_moc/manifest-migration.md` (신규)
- `_moc/plugins-catalog.md` (신규)
- `_moc/discord-ci-strategy.md` (신규)
- `_moc/engineering-knowledge-surface.md` (신규)
- `_moc/hermes-yule-integration.md` (신규)
- `_moc/issue-73-tech-lead-runtime.md` (신규)

### 5. broken backlinks 정리

- 그래프 stub (실제 파일 없음) 4 개를 새 노트 / 통합 대상으로
  redirect: `2026-05-09_issue-73-{decision|research}-tech-lead-runtime-loop`
  → `task-log-tech-lead-runtime-loop-issue-73`, `2026-05-08_issue-69-*`
  → `decision-engineering-knowledge-share-boundary`

## 결정 요약

- **파일명에서 날짜 제거** — 버전 / 작성일은 frontmatter `created_at` +
  본문 첫 표 만으로 관리. [[link]] 안정성 + 사용자 그래프 가독성 둘 다
  확보.
- **dash 분리자 통일** — 이미 `filename_convention.validate_filename`
  이 검증하던 컨벤션과 맞춤 (underscore 는 mistake_ledger 에 등록될
  hard rail 위반).
- **폴더별 README 가 hub** — kind 기준 (decisions / research /
  task-logs / knowledge) + 주제 기준 (`_moc/` 의 6 hub) 이 양축.

## 비결정 (의도적)

- vault root 의 0-line stub (`/Users/masterway/local-dev/
  yule-agent-vault/2026-05-08_issue-69-decision-engineering-agent-
  authoring-policy.md`) 삭제는 안전장치로 거부됨. 사용자가 직접 삭제.
- 기존 노트 본문의 broken [[link]] 문장 수정은 안전장치 거부 — 본문
  복원은 사용자 검토 영역.

## 다음 행동

1. 사용자가 vault 에서 git commit/push (별도 git repo)
2. repo `policies/runtime/vault/naming-convention.md` 갱신
   (날짜 제거 + dash + 본문 표 양식 반영)
3. F15 잔여 commit (hr/finance/sales-cs/legal 부서 골격 + PM skills +
   governance 테스트)

## 회귀

- `pytest tests/obsidian tests/engineering tests/discord` →
  **1772 passed in 3.45s**
