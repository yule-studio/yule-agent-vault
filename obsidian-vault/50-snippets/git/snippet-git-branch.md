---
title: "git branch / switch — 브랜치 관리"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:20:00+09:00
tags:
  - snippet
  - git
  - branch
---

# git branch / switch — 브랜치 관리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
브랜치 생성 / 이동 / 정리 — 모든 흐름의 출발점.

## 코드

```bash
# 1. 보기
git branch                     # 로컬
git branch -a                  # 원격 포함
git branch -r                  # 원격만
git branch -vv                 # 마지막 commit + tracking branch
git branch --merged main       # main 에 머지된 브랜치 (정리 후보)
git branch --no-merged main    # 아직 안 머지된

# 2. 만들기
git branch feature/x           # 만 들기만 (이동 X)
git switch -c feature/x        # 만들고 이동 (신문법 ★)
git checkout -b feature/x      # 구문법

# 3. 특정 base 에서 만들기
git switch -c feature/x main          # main 기반
git switch -c hotfix/y v1.2.0         # 태그 기반
git switch -c bug/z abc123            # 특정 commit 기반

# 4. 이동
git switch main                # 신문법
git checkout main              # 구문법
git switch -                   # 직전 브랜치로

# 5. 이름 바꾸기
git branch -m old-name new-name
git branch -m new-name         # 현재 브랜치 이름 변경

# 6. 삭제
git branch -d feature/x        # 안전 (머지 안 됐으면 거부)
git branch -D feature/x        # 강제 (위험)

# 7. 원격 브랜치
git fetch origin                              # 원격 정보 갱신
git switch -c feature/x origin/feature/x      # 원격 → 로컬 추적
git switch feature/x                          # 원격에만 있으면 자동 추적 (Git 2.23+)
git push origin --delete feature/x            # 원격 브랜치 삭제

# 8. 머지된 브랜치 일괄 정리
git branch --merged main | grep -v 'main\|master\|\*' | xargs git branch -d

# 9. detached HEAD (특정 commit 만 보기)
git switch --detach abc123
```

## 사용법
- 새 작업 = `git switch -c feature/<topic>` 부터
- 정리 = `--merged main` + `xargs -d`

## 주의
- `-D` 는 머지 안 된 commit 영구 손실 — `git reflog` 로 복구 가능하지만 30 일 한도
- `switch` 는 working dir 의 변경이 충돌하면 거부 → stash 후 이동

## 관련
- [[snippet-git-merge-rebase]] — 브랜치 합치기
- [[snippet-git-worktree]] — 여러 브랜치 동시 작업
