---
title: "git reflog — 잃어버린 commit 복구"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:05:00+09:00
tags:
  - snippet
  - git
  - reflog
  - recovery
---

# git reflog — 잃어버린 commit 복구

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
`reset --hard` / `rebase` / 잘못된 `branch -D` 후의 작업을 살린다. **로컬의 마지막 안전망**.

## 코드

```bash
# 1. 모든 HEAD 이동 기록
git reflog
# HEAD@{0}: reset: moving to abc123
# HEAD@{1}: commit: ...
# HEAD@{2}: rebase finished: ...
# ...

# 2. 특정 브랜치의 reflog
git reflog show main
git reflog show feature/x

# 3. 잘못된 reset 되돌리기
git reset --hard HEAD@{1}              # 직전 상태로
git reset --hard HEAD@{2.hours.ago}    # 시간 기반

# 4. 삭제한 브랜치 복구
git reflog                             # 그 브랜치의 마지막 commit hash 찾기
git branch feature/x abc123            # 복구
# 또는
git reflog --all | grep feature/x

# 5. interactive rebase 후 망가졌을 때
git reflog                             # rebase 전 HEAD 확인
git reset --hard <pre-rebase-hash>

# 6. stash 의 reflog
git stash list                         # 보이는 stash
git reflog show stash                  # 옛 drop 한 stash 도 찾을 수 있음

# 7. 어디서도 reference 없는 commit 찾기
git fsck --lost-found
git fsck --unreachable
# 출력에서 hash 로 commit 내용 확인:
git show <hash>

# 8. reflog 보존 기간
git config gc.reflogExpire             # 기본 90 일
git config gc.reflogExpireUnreachable  # 기본 30 일
git config --global gc.reflogExpire 365.days
```

## 사용법
- "방금 `reset --hard` 한 거 아니야?!" = `git reflog` → `git reset --hard HEAD@{1}`
- 실수한 force push 후 = 로컬 reflog → 원격에 다시 push

## 주의
- reflog 는 **로컬에만** — clone / push 와 무관
- 30~90 일 후 garbage collect — 영구 손실 가능
- 다른 머신에는 reflog 없음

## 관련
- [[snippet-git-undo]] — reset 의 결과를 되돌릴 때
- [[snippet-git-amend-rewrite]] — rebase 실수 복구
