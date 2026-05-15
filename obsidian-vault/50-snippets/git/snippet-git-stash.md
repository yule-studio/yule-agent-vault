---
title: "git stash — 작업 잠시 치우기"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:30:00+09:00
tags:
  - snippet
  - git
  - stash
---

# git stash — 작업 잠시 치우기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"커밋하기엔 안 끝났는데, 다른 브랜치로 가야 할 때" 작업을 임시 보관.

## 코드

```bash
# 1. 기본 저장 (tracked 의 변경만)
git stash
git stash push -m "WIP: api refactor"   # 메시지 권장

# 2. untracked 도 포함
git stash -u                   # --include-untracked
git stash -a                   # --all (ignored 도)

# 3. 일부 파일만 stash
git stash push -m "css only" -- src/styles/
git stash push -p              # patch — hunk 별 선택

# 4. 보기
git stash list                 # stash@{0}: ..., stash@{1}: ...
git stash show stash@{0}       # stat
git stash show -p stash@{0}    # patch (실제 diff)

# 5. 적용
git stash pop                  # 가장 위 stash 적용 + 제거
git stash apply stash@{1}      # 적용만 (stash 유지)
git stash pop --index          # 스테이지 상태까지 복원

# 6. 삭제
git stash drop stash@{0}       # 특정 하나
git stash clear                # 전부 (위험)

# 7. stash 를 브랜치로 (충돌이 복잡할 때)
git stash branch fix/wip stash@{1}
# 새 브랜치를 stash 시점에 만들고 stash 적용 + 삭제

# 8. 충돌 후 stash 복구
# pop 이 충돌 → stash 가 사라지지 않음 (apply 와 같은 효과)
# 해결 후: git stash drop 으로 명시 삭제
```

## 사용법
- 긴급한 hotfix 호출 = stash → switch → fix → switch back → pop
- 같은 변경을 여러 브랜치에서 시도 = stash apply 반복

## 주의
- stash 는 로컬에만 — 다른 머신으로 못 옮김 (patch 파일로 export 필요)
- 오래된 stash 는 `git fsck --unreachable` 외에는 안 보임
- `pop` 후 충돌 = stash 가 안 사라짐 (재시도 가능, 그러나 신중)

## 관련
- [[snippet-git-undo]] — restore / reset 으로 대체 가능한 경우
- [[snippet-git-branch]] — switch 전 정리
