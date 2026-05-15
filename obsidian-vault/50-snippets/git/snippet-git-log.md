---
title: "git log — 히스토리 / timeline"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:10:00+09:00
tags:
  - snippet
  - git
  - log
  - timeline
---

# git log — 히스토리 / timeline

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"언제 / 누가 / 무엇을 / 왜" 를 시간순으로 본다.

## 코드

```bash
# 1. 기본 — 최근 commit 순
git log
git log -n 10                  # 최근 10개만

# 2. 한 줄 요약 (가장 자주 쓰는)
git log --oneline
git log --oneline -20

# 3. graph (브랜치 시각화)
git log --oneline --graph --all --decorate

# 4. 특정 파일의 history
git log -- src/api.py
git log -p -- src/api.py       # 각 commit 의 patch 도

# 5. 한 줄의 history (line-level blame)
git log -L 10,20:src/api.py    # api.py 의 10~20 줄 변화
git log -L :function_name:src/api.py

# 6. 저자 / 메시지 필터
git log --author="masterway"
git log --grep="bug"           # commit message 안 "bug"
git log --grep="fix" --grep="bug" --all-match   # 두 단어 동시

# 7. 시간 필터
git log --since="2026-05-01" --until="2026-05-15"
git log --since="2 weeks ago"

# 8. 변경 통계
git log --stat                 # 파일 별 +/-
git log --shortstat
git log --numstat              # 숫자만 (스크립트 친화)

# 9. branch / range
git log main..feature/x        # feature/x 에만 있는 commit
git log feature/x ^main        # 같음
git log --all --not main       # main 외 모든 브랜치의 unique commit

# 10. 예쁘게 + 색
git log --pretty=format:'%C(yellow)%h %C(cyan)%an %C(green)%ar%Creset %s' --graph
```

## 사용법
- 평소 alias: `git lg` → graph + oneline + author + 시간 (config 노트 참조)
- PR 직전: `git log --oneline main..HEAD` 로 commit 흐름 확인

## 주의
- `git log A..B` = A 의 조상이 **아닌** B 의 조상 (B 에만 있는 것)
- 시간 필터는 commit time / author time 둘 다 영향 — `--committer-date` 추가 사용

## 관련
- [[snippet-git-diff]] — log 본 후 commit 사이 변경 보기
- [[snippet-git-blame]] — 라인 단위 책임
- [[snippet-git-search]] — message 가 아니라 코드로 찾기
