---
title: "git undo — 되돌리기 (restore / reset / revert)"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:35:00+09:00
tags:
  - snippet
  - git
  - undo
  - reset
  - revert
---

# git undo — restore / reset / revert

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"되돌리기" 는 3 영역 (working / staged / commit) 에 따라 명령어가 다르다.

## 코드

```bash
# === working dir 의 변경 되돌리기 (commit 안 한 것)
git restore file.py            # 신문법 — file.py 를 HEAD 로 복원
git restore .                  # 전체 (위험)
git checkout -- file.py        # 구문법

# === staged 만 풀기 (변경은 유지)
git restore --staged file.py
git reset HEAD file.py         # 구문법

# === 마지막 commit 되돌리기 — 변경은 유지 (가장 자주)
git reset HEAD~                # = --mixed, 변경은 working 으로
git reset --soft HEAD~         # 변경은 staged 로 유지
git reset --hard HEAD~         # 변경 자체를 버림 (위험)

# === 여러 commit 되돌리기
git reset --soft HEAD~3        # 최근 3개 → 다시 스테이지로
git reset --hard origin/main   # 로컬을 원격 그대로 (커밋 손실)

# === 이미 push 한 commit 되돌리기 (안전 — 새 commit 추가)
git revert <commit-hash>       # 해당 commit 의 역 변경을 새 commit 으로
git revert HEAD                # 마지막 commit 취소
git revert --no-commit HEAD~3..HEAD   # 3 commit 일괄 revert (staged 만)
git revert -m 1 <merge-commit> # merge commit revert (parent 1 선택)

# === 파일 단위 — 옛 버전 가져오기
git restore --source=<commit> file.py
git checkout <commit> -- file.py

# === 잘못 reset 했을 때
git reflog                     # 모든 HEAD 이동 기록
git reset --hard HEAD@{1}      # 직전 상태로 복구

# === clean — untracked 도 정리
git clean -n                   # 미리보기 (dry-run)
git clean -fd                  # 강제 (위험, 복구 불가)
```

## 사용법
- **로컬만** 의 commit 되돌리기 → `reset`
- **이미 공유한** commit 되돌리기 → `revert` (history 안 바뀜)
- 단순 파일 복원 → `restore` (신문법, 명확)

## 주의
- `reset --hard` + `clean -fd` = **복구 불가능** (untracked 는 reflog 안 잡힘)
- merge commit 의 revert = parent 명시 (`-m 1`)
- revert 한 변경을 나중에 다시 merge 하려면 추가 작업 필요 (revert the revert)

## 관련
- [[snippet-git-reflog]] — 잃어버린 commit 찾기
- [[snippet-git-stash]] — 변경 임시 보존하며 되돌리기
- [[snippet-git-clean]] — untracked 정리
