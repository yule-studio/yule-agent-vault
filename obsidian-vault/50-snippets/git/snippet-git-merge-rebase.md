---
title: "git merge / rebase — 브랜치 합치기"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:25:00+09:00
tags:
  - snippet
  - git
  - merge
  - rebase
---

# git merge / rebase — 브랜치 합치기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
다른 브랜치의 변경을 가져와 합친다 — merge / rebase 둘 중 어느 것을 쓸지가 핵심.

## 코드

```bash
# 1. 기본 merge (feature → main)
git switch main
git pull --ff-only             # 원격 최신화
git merge --no-ff feature/x    # merge commit 강제 (history 보존)
git merge --ff-only feature/x  # fast-forward 만 (선형 history)

# 2. rebase (feature 의 base 를 main 최신으로)
git switch feature/x
git fetch origin
git rebase origin/main         # 또는 git rebase main

# 3. interactive rebase (commit 정리)
git rebase -i HEAD~5           # 최근 5개 commit 의 pick/squash/reword/drop
# pick   = 그대로
# reword = 메시지 변경
# squash = 직전 commit 과 합침 (메시지 합침)
# fixup  = squash + 메시지 버림
# drop   = 제거

# 4. rebase 도중 충돌
# 충돌 파일 수정
git add <resolved>
git rebase --continue          # 계속
git rebase --skip              # 이 commit 건너뜀
git rebase --abort             # 처음으로 되돌림

# 5. merge 도중 충돌
git merge --abort

# 6. 두 브랜치 공통 조상
git merge-base main feature/x

# 7. pull = fetch + merge / rebase
git pull                       # 기본 merge
git pull --rebase              # rebase 로 (선형 history)
git config --global pull.rebase true   # 영구 rebase 모드

# 8. squash merge (PR squash 같은 효과)
git merge --squash feature/x
git commit -m "feat: ..."      # 한 commit 으로

# 9. rerere — 충돌 해결 기억해서 재사용
git config --global rerere.enabled true
```

## 사용법
- 공유된 브랜치 (main / develop) → **merge** (history 보존)
- 내 작업 브랜치 → **rebase** (선형, 깔끔)
- 옛 PR 의 commit 정리 → `git rebase -i`

## 주의
- **이미 push 한 브랜치 rebase 금지** — force push 필요 → 동료 작업 손실
- merge commit = "어떻게 합쳤나" 보존, rebase = "왜 이렇게 했나" 의 흐름만 보존
- 회사 정책에 squash / rebase / merge 셋 중 하나 정해진 경우 많음

## 관련
- [[snippet-git-amend-rewrite]] — interactive rebase 자세히
- [[snippet-git-remote]] — push --force-with-lease
