---
title: "git diff — 변경 확인"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:05:00+09:00
tags:
  - snippet
  - git
  - diff
---

# git diff — 변경 확인

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"내가 / 우리가 뭘 바꿨나" 를 영역별로 빠르게 본다.

## 코드

```bash
# 1. 아직 스테이지 안 한 변경 (working ↔ index)
git diff

# 2. 스테이지된 변경 (index ↔ HEAD) — commit 전 확인용
git diff --staged              # 또는 --cached

# 3. working + staged 합쳐서 HEAD 와 비교
git diff HEAD

# 4. 두 commit 사이
git diff <old>..<new>
git diff abc123..HEAD          # abc123 → 지금까지의 차이

# 5. 두 브랜치 사이
git diff main..feature/x       # main 에 없고 feature/x 에 있는 것
git diff main...feature/x      # ... = 두 브랜치 공통 분기점부터의 차이 (PR view)

# 6. 특정 파일 / 디렉토리만
git diff -- src/api.py
git diff main..HEAD -- src/

# 7. word-level diff (긴 줄의 한 글자 변경)
git diff --word-diff

# 8. stat 만 (어떤 파일이 얼마나 변했나)
git diff --stat
git diff main..HEAD --stat

# 9. 변경된 파일 이름만
git diff --name-only main..HEAD

# 10. 공백 무시
git diff -w                    # 모든 공백 무시
git diff --ignore-blank-lines
```

## 사용법
- PR 만들기 전 = `git diff main...HEAD --stat` 로 큰 그림 → `git diff main...HEAD` 로 자세히
- 리뷰 요청 받은 변경 확인 = `git diff <base>..<head>`

## 주의
- `..` vs `...` 차이 — `...` 는 merge base 기준, GitHub PR diff 와 같음
- `git diff HEAD~3..HEAD` 의 `HEAD~3` 은 **3 commit 전**, `HEAD^3` 은 **3rd parent** (merge 의)
- 바이너리 파일은 diff 안 보임 → `--text` 강제 (위험)

## 관련
- [[snippet-git-log]] — 어느 commit 사이 diff 볼지 결정
- [[snippet-git-search]] — diff 안의 문자열로 찾기
