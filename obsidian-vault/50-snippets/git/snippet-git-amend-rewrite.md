---
title: "git amend / rebase -i — commit 수정"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:50:00+09:00
tags:
  - snippet
  - git
  - amend
  - rebase
  - rewrite
---

# git amend / rebase -i — commit 수정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
이미 만든 commit 을 수정 — 메시지 / 내용 / 합치기 / 나누기.

## 코드

```bash
# 1. 직전 commit 수정 (메시지)
git commit --amend                   # editor 열림
git commit --amend -m "새 메시지"

# 2. 직전 commit 에 파일 추가 (메시지 그대로)
git add forgot.py
git commit --amend --no-edit

# 3. 작성자 변경 (실수로 다른 이메일로 commit)
git commit --amend --author="이름 <email@x.com>"

# 4. 여러 commit 정리 (interactive rebase)
git rebase -i HEAD~5                 # 최근 5개
# 편집기에서:
#   pick   abc 첫 commit
#   reword def 두 번째 → 메시지 변경
#   squash ghi 세 번째 → 위 commit 과 합침 (메시지 합침)
#   fixup  jkl 네 번째 → squash + 메시지 버림
#   edit   mno 다섯 번째 → 멈춰서 수정 가능
#   drop   pqr 여섯 번째 → 제거
#   reorder = 줄 순서 바꾸기

# 5. rebase -i 도중 edit 으로 멈췄을 때
git commit --amend                   # 수정
git rebase --continue
# 또는 변경을 더 추가:
echo "x" >> file
git add file
git commit --amend --no-edit
git rebase --continue

# 6. 한 commit 을 두 commit 으로 나누기
git rebase -i HEAD~3
# 분할할 commit 을 `edit` 로 변경
# rebase 가 멈추면:
git reset HEAD~                      # 변경을 working 으로
git add part1.py && git commit -m "part 1"
git add part2.py && git commit -m "part 2"
git rebase --continue

# 7. 옛 commit 에 변경 끼우기 (fixup + autosquash)
git commit --fixup=<commit>          # "fixup! ..." 메시지로 commit
git rebase -i --autosquash <base>    # 자동으로 fixup 자리 잡음

# 8. amend / rewrite 후 push
git push --force-with-lease          # ★ 안전한 force
# 절대 X: git push --force (남의 commit 덮어쓸 수 있음)
```

## 사용법
- PR 리뷰 코멘트 반영 = `--fixup` + `--autosquash` 로 깔끔히
- 큰 commit 을 작은 단위로 = `edit` + `reset HEAD~` + 부분 add

## 주의
- **이미 다른 사람이 fetch 한 commit 의 rewrite = 금지** (force push 분쟁)
- `--amend` 는 새 hash 를 만든다 — 같은 commit 이 아님
- rebase 도중 충돌 → `--abort` 로 언제든 처음으로

## 관련
- [[snippet-git-merge-rebase]] — interactive rebase 자세히
- [[snippet-git-reflog]] — 잘못된 rebase 복구
