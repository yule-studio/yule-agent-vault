---
title: "git remote / fetch / push / pull"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:45:00+09:00
tags:
  - snippet
  - git
  - remote
  - push
  - pull
---

# git remote — 원격과 주고받기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
로컬 ↔ 원격 동기화 — fetch / pull / push 의 정확한 통제.

## 코드

```bash
# 1. remote 확인 / 관리
git remote -v
git remote add origin git@github.com:org/repo.git
git remote set-url origin git@github.com:org/repo.git   # URL 변경
git remote rename origin upstream
git remote remove origin

# 2. fetch — 원격 가져오기만 (working 영향 X)
git fetch origin
git fetch --all                # 모든 remote
git fetch --prune              # 원격에서 삭제된 브랜치 로컬 정리 (★)
git fetch origin main:main     # 로컬 main 도 ff (안 체크아웃)

# 3. push
git push origin feature/x
git push -u origin feature/x   # tracking 설정 (이후 인자 없이 push 가능)
git push                       # tracking 있으면 충분

# 4. push 의 안전 옵션
git push --force-with-lease    # 내가 받은 last 가 여전히 원격일 때만 (★)
git push --force               # 위험 — 동료 작업 덮어쓸 수 있음

# 5. 모든 브랜치 / 태그 push
git push --all origin
git push --tags
git push origin :feature/x     # 원격 브랜치 삭제 (= --delete)
git push origin --delete feature/x

# 6. pull
git pull                       # = fetch + merge
git pull --rebase              # = fetch + rebase
git pull --ff-only             # fast-forward 만 (안전)
git config --global pull.ff only      # 영구 ff-only
git config --global pull.rebase true  # 영구 rebase

# 7. 원격 브랜치 → 로컬 추적
git fetch origin
git switch feature/x           # 자동 추적 (Git 2.23+)
git branch --set-upstream-to=origin/feature/x  # 수동

# 8. 원격과의 차이
git log origin/main..HEAD      # 내가 push 안 한 commit
git log HEAD..origin/main      # 내가 pull 안 한 commit
git status -sb                 # `ahead 2, behind 1` 한 줄
```

## 사용법
- rebase 후 push = `--force-with-lease` (절대 `--force` X)
- 원격에 없는 브랜치 정리 = `git fetch --prune`

## 주의
- `git push --force` = main 에 절대 X. 동료 작업 손실 가능
- `git pull` 의 기본 동작은 환경마다 다름 → `pull.rebase` 또는 `pull.ff` 명시 권장
- credential 은 `git config credential.helper` 로 (HTTPS 인 경우)

## 관련
- [[snippet-git-amend-rewrite]] — rewrite 후 push 의 안전
- [[snippet-git-merge-rebase]] — pull --rebase 의 동작
