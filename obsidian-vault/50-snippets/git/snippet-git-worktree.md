---
title: "git worktree — 여러 브랜치 동시 작업"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:15:00+09:00
tags:
  - snippet
  - git
  - worktree
---

# git worktree — 여러 브랜치 동시 작업

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
한 repo 에서 **여러 디렉토리에 다른 브랜치 동시 체크아웃** — clone 여러 번 안 해도 됨.

## 코드

```bash
# 1. 추가
git worktree add ../repo-feature feature/x
# → ../repo-feature 디렉토리에 feature/x 가 checkout 된 워크트리

# 2. 새 브랜치를 동시에 만들기
git worktree add -b hotfix/y ../repo-hotfix main
# → main 에서 hotfix/y 브랜치 만들고 ../repo-hotfix 에 체크아웃

# 3. 보기
git worktree list
# /Users/me/repo            abc123 [main]
# /Users/me/repo-feature    def456 [feature/x]
# /Users/me/repo-hotfix     ghi789 [hotfix/y]

# 4. 제거
git worktree remove ../repo-feature       # 디렉토리도 같이 삭제
git worktree remove -f ../repo-feature    # 강제 (uncommitted 있어도)

# 5. lock / unlock (외장 디스크 등 분리될 때)
git worktree lock ../repo-feature
git worktree unlock ../repo-feature

# 6. 청소 (사라진 워크트리의 메타 정리)
git worktree prune

# 7. detached HEAD 워크트리 (release tag 검토용)
git worktree add --detach ../repo-v1.2 v1.2.0

# 8. bare repo + 여러 워크트리 (서버 / 자동화)
git clone --bare git@x.com:repo.git repo.git
cd repo.git
git worktree add ../main main
git worktree add ../staging staging
```

## 사용법
- main 에서 작업 중 hotfix 호출 = `worktree add ../repo-hotfix main` → stash 없이 즉시 시작
- PR 리뷰 = 새 worktree 로 checkout → 본인 작업 영향 X
- AI 에이전트 병렬 작업 = 각자의 worktree

## 주의
- 같은 브랜치를 두 워크트리에서 체크아웃 못 함 (Git 2.5+ 의 제약)
- 워크트리 디렉토리 그냥 삭제 시 메타 남음 → `git worktree prune`
- `.git` 은 메인에만, 워크트리는 `.git` 파일 (포인터)

## 관련
- [[snippet-git-stash]] — 한 워크트리 안에서 임시 보관
- [[snippet-git-branch]] — 브랜치 자체 관리
