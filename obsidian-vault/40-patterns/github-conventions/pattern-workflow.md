---
title: "이슈 → PR → Merge 워크플로우"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:55:00+09:00
tags:
  - github
  - workflow
  - process
---

# 이슈 → PR → Merge 워크플로우

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 표준 흐름 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 전체 흐름

```
[1] 이슈 생성 (템플릿)
    ↓
[2] 라벨 부여 → priority 계산
    ↓
[3] 작업 분배 (assignee)
    ↓
[4] 브랜치 생성 (이슈번호 포함)
    ↓
[5] 작업 + commit (의미 단위)
    ↓
[6] PR 생성 (draft → ready)
    ↓
[7] CI 검증 + review
    ↓
[8] 머지 (이슈 자동 close)
    ↓
[9] release (필요 시)
```

---

## 2. [1] 이슈 생성

GitHub repo 의 `New Issue` → 6 종 템플릿 중 선택:
- [[pattern-issue-bug-report|🐞 Bug Report]]
- [[pattern-issue-feature|✨ Feature Request]]
- [[pattern-issue-cicd|🛠 CI/CD Config]]
- [[pattern-issue-docs|📑 Docs Request]]
- [[pattern-issue-release|🚀 Release]]
- [[pattern-issue-task|[TASK] Task]]

---

## 3. [2] 라벨 → Priority

자세히 → [[pattern-label-rules]]

```
priority = base + Σ label.priority_boost
```

자동 라벨링 (선택):
- 파일 path 기반 (`actions/labeler`)
- title prefix 기반
- 본문 keyword 기반

수동:
- triage 단계에서 maintainer 가 검토

---

## 4. [3] 작업 분배

| 방법 | |
| --- | --- |
| 수동 | maintainer 가 GitHub `assignees` |
| 자동 | 에이전트가 priority 내림차순 + role 매칭 |
| self | 기여자가 self-assign |

사용자 메모리 (`feedback_auto_merge_authorization`):
> "사용자가 #81 후속 F1~F8 사이클에 한해 self-merge 권한 부여 (2026-05-11). CI+회귀 OK 시 auto-merge OK."

→ 특정 사이클 / 라벨 / 작업 종류에 따라 에이전트 self-merge 가능.

---

## 5. [4] 브랜치 생성

### 5.1 네이밍

```
{type}/{issue-num}-{short-kebab}

예:
  feature/123-csv-export
  bugfix/124-login-500
  refactor/125-extract-service
  cicd/126-deploy-staging
  docs/127-branch-strategy
  release/128-v1.2.3
  task/129-setup-eslint
```

### 5.2 base branch

| 종류 | base |
| --- | --- |
| 일반 feature | `develop` (또는 `main` if trunk-based) |
| hotfix | `main` |
| release | `develop` → `release/vX.Y.Z` → `main` |

---

## 6. [5] 작업 + Commit

### 6.1 Commit 메시지 (사용자 메모리 `feedback_commit_format`)

```
{이모지} #{이슈번호} {유형} commit {N} — {짧은 제목}

변경 이유:
- ...

주요 변경 사항:
- ...

비고:
- ...
```

⚠️ **Co-Authored-By 절대 X**.

이모지 / 유형 매핑:
- ✨ #N feature commit 1
- 🐛 #N fix commit 1
- ♻️ #N refactor commit 1
- 📝 #N docs commit 1
- ✅ #N test commit 1
- 🔧 #N chore commit 1
- 🚀 #N deploy commit 1

### 6.2 Commit 분리 정책

사용자 메모리 (`feedback_commit_splitting_policy`):
- PR 당 **최소 3 commit**
- squash 는 docs-only 1-commit chore 에만
- `--merge` / `--rebase` 우선

---

## 7. [6] PR 생성

자세히 → [[pattern-pr-template]]

### 7.1 Draft → Ready

```bash
gh pr create --draft --title "feat: CSV export (#123)" --body-file pr-body.md
# 작업 완료 후
gh pr ready
```

### 7.2 PR 제목

```
{type}: {short-summary} (#{issue-num})

예:
  feat: CSV export (#123)
  fix: 로그인 500 에러 (#124)
  docs: 브랜치 전략 (#127)
```

### 7.3 본문 — 이슈 link

```markdown
## 📌 관련 이슈

closed #123
```

`closed #N` 키워드가 머지 시 이슈 자동 close.

---

## 8. [7] CI + Review

### 8.1 자동 검증
- build / test
- lint / format
- security scan
- coverage
- preview deploy (선택)

### 8.2 Review
- 최소 1 명 approval
- CODEOWNERS 자동 reviewer
- 모든 must-fix 해결 + nit 응답

자세히 → [[pattern-pr-template#6-review-가이드]]

---

## 9. [8] 머지

### 9.1 머지 방식

| 방식 | 언제 |
| --- | --- |
| **Merge commit** | 일반 — feature 의 commit history 보존 |
| **Squash** | docs-only 1-commit chore |
| **Rebase** | clean history 원할 때 (drawback: lost context) |

### 9.2 이슈 자동 close
PR body 의 `closed #N` 키워드 → 머지 시 자동.

### 9.3 브랜치 정리
머지 후 즉시 브랜치 삭제 (GitHub 의 "Delete branch" 자동 옵션).

---

## 10. [9] Release (필요 시)

자세히 → [[pattern-issue-release]]

```bash
# 1. CHANGELOG 갱신
gh pr list --search "merged:>2026-05-01" --json title,number > /tmp/changes.json

# 2. tag
git tag vX.Y.Z
git push origin vX.Y.Z

# 3. GitHub Release
gh release create vX.Y.Z --notes-from-tag

# 4. 자동 배포 확인 + smoke test
```

---

## 11. 에이전트 자동화

### 11.1 RAG/CAG 입력
이 폴더 (`github-conventions/`) 의 모든 노트 = 에이전트의 컨텍스트.

### 11.2 에이전트 의사결정 흐름

```
1. backlog fetch (GitHub API)
2. priority 계산 ([[pattern-label-rules]])
3. role 매칭 (라벨 → 어느 에이전트?)
   - 🐞 bugfix + 📬 api → engineering-agent
   - 📑 docs → docs-agent
   - 🌏 deploy → devops-agent
4. 작업 (브랜치 + commit + PR)
5. self-merge 권한 확인 (사용자 메모리 `feedback_auto_merge_authorization`, .claude 안)
6. CI + 회귀 OK → merge
```

### 11.3 안전장치
- PR 당 commit 수 검증 (≥ 3)
- CI 모두 green
- 사용자 메모리의 권한 범위 안인지
- 영향 범위 (변경 파일 수 / 크기) 한계

---

## 12. 자주 보는 변형

### 12.1 hotfix
```
main → bugfix/Nm-hot-... → main (직접) + develop (cherry-pick)
```

### 12.2 trunk-based
`develop` 없음. 모든 PR `main` 으로. feature flag.

### 12.3 GitFlow
`develop` + `release/*` + `hotfix/*`. 복잡하지만 multi-release 환경.

---

## 13. 관련

- [[github-conventions]] — hub
- [[pattern-label-rules]] — priority
- [[pattern-pr-template]] — PR 본문
- [[pattern-issue-bug-report]] / [[pattern-issue-feature]] / ... — 6 종 이슈
- 사용자 메모리 (.claude/projects/.../memory/):
  - `feedback_commit_format`
  - `feedback_commit_splitting_policy`
  - `feedback_auto_merge_authorization`
  - `feedback_github_templates_required`
