---
title: "git — Snippet Hub"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:00:00+09:00
tags:
  - snippet
  - git
  - hub
---

# git — Snippet Hub

**[[../snippets|↑ 50-snippets]]**

> 자주 다시 쓰는 git 명령어 모음. **상황별** 분류 — 명령어 이름이 아니라
> "이럴 때 뭘 친다" 기준.

## 상황 → 명령어

| 상황 | 노트 | 핵심 명령 |
| --- | --- | --- |
| 지금 뭐가 바뀌었나? | [[snippet-git-diff]] | `git diff`, `git diff --staged` |
| 누가 / 언제 / 왜 바꿨나? | [[snippet-git-log]] | `git log`, `git log --graph` |
| 한 줄 누가 만들었나? | [[snippet-git-blame]] | `git blame`, `git log -L` |
| 스테이지 / 커밋 준비 | [[snippet-git-status-stage]] | `git status`, `git add -p` |
| 브랜치 만들기 / 옮기기 | [[snippet-git-branch]] | `git switch`, `git branch` |
| 두 브랜치 합치기 | [[snippet-git-merge-rebase]] | `git merge`, `git rebase` |
| 작업 잠시 치워두기 | [[snippet-git-stash]] | `git stash push`, `git stash pop` |
| 방금 한 거 되돌리기 | [[snippet-git-undo]] | `git restore`, `git reset`, `git revert` |
| 다른 브랜치의 한 commit 만 가져오기 | [[snippet-git-cherry-pick]] | `git cherry-pick` |
| 원격과 주고받기 | [[snippet-git-remote]] | `git fetch`, `git push`, `git pull` |
| commit 메시지 / 내용 수정 | [[snippet-git-amend-rewrite]] | `git commit --amend`, `git rebase -i` |
| 어떤 문자열 / 함수 언제 들어왔나? | [[snippet-git-search]] | `git log -S`, `git log -G` |
| 잃어버린 commit 복구 | [[snippet-git-reflog]] | `git reflog`, `git fsck` |
| 릴리스 / 버전 마킹 | [[snippet-git-tag]] | `git tag -a` |
| 여러 브랜치 동시 열기 | [[snippet-git-worktree]] | `git worktree add` |
| 워크트리 청소 | [[snippet-git-clean]] | `git clean`, `git restore` |
| 버그 도입 commit 찾기 | [[snippet-git-bisect]] | `git bisect` |
| 자주 쓰는 alias / 설정 | [[snippet-git-config-alias]] | `git config --global alias` |

## 큰 그림: 3개 영역

```
working dir   →  index (staged)  →  repository (commit)
   workspace        staging              history
       ↑               ↑                     ↑
   restore        restore --staged       reset / revert
```

→ "어디" 의 변경을 다루느냐로 명령어가 갈린다.

## 관련

- [[../snippets|↑ 50-snippets]]
- [[../../40-patterns/github-conventions/github-conventions]] — 브랜치 / 커밋 컨벤션
- [[../../40-patterns/git-conflict/git-conflict]] — 충돌 대비 / 해결 패턴 (백업·머지용·해결용 브랜치 분리)
