---
title: "git cherry-pick — 특정 commit 만 가져오기"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:40:00+09:00
tags:
  - snippet
  - git
  - cherry-pick
---

# git cherry-pick

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
다른 브랜치의 **특정 commit 하나만** (또는 몇 개만) 현재 브랜치로 옮긴다. hotfix backport 의 핵심.

## 코드

```bash
# 1. 단일 commit 가져오기
git switch release/v1.2
git cherry-pick abc123                  # 그 commit 을 여기에 적용

# 2. 여러 commit 한번에
git cherry-pick abc123 def456 ghi789    # 순서대로
git cherry-pick abc123..def456          # 범위 (abc123 다음부터 def456 까지)
git cherry-pick abc123^..def456         # abc123 도 포함

# 3. 메시지 안 commit 까지 (staged 만)
git cherry-pick -n abc123               # --no-commit
# → 여러 변경 합쳐서 한 commit 으로 만들 때

# 4. 메시지 수정하면서
git cherry-pick -e abc123               # 편집기 열림

# 5. signed-off / origin 표시
git cherry-pick -x abc123               # 메시지에 "cherry picked from ..." 추가

# 6. 충돌 해결
# 충돌 발생 → 파일 수정 → 
git add <file>
git cherry-pick --continue
# 또는
git cherry-pick --abort                 # 처음으로
git cherry-pick --skip                  # 이 commit 건너뛰기

# 7. merge commit cherry-pick
git cherry-pick -m 1 <merge-commit>     # parent 1 기준의 diff

# 8. range — 한 줄로 hotfix 묶음 이동
git cherry-pick main~5..main~2          # main 의 일부 구간만
```

## 사용법
- hotfix backport: `main` 의 bug fix commit → `release/v1.x` 로
- 같은 일을 두 브랜치에 = 한 commit 작성 → 두 브랜치에 cherry-pick

## 주의
- 같은 변경의 **새 commit** 이 생긴다 — hash 다름. 나중에 같은 변경이 또 머지되면 충돌
- `cherry-pick` 남발 = history 망가짐 → 가능하면 base 브랜치에 한 번만 commit + branching off
- merge commit cherry-pick 은 신중 (`-m`)

## 관련
- [[snippet-git-merge-rebase]] — 보통은 rebase 가 더 맞음
- [[snippet-git-revert]] — 비슷한 mechanic 의 반대
