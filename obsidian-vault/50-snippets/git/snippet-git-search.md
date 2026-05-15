---
title: "git search — 코드 / commit 안에서 찾기"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:55:00+09:00
tags:
  - snippet
  - git
  - search
  - pickaxe
  - grep
---

# git search — 코드 / commit 안에서 찾기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"이 함수 / 문자열 언제 들어왔고 / 언제 빠졌지?" — pickaxe (`-S`/`-G`) 와 `git grep`.

## 코드

```bash
# 1. 어떤 commit 에서 "FOO_BAR" 가 추가/제거 됐나? (★ pickaxe)
git log -S 'FOO_BAR'                 # 등장 횟수가 바뀐 commit
git log -S 'def my_func'             # 함수 정의가 들어오거나 빠진 시점
git log -S 'API_KEY' --all -p        # 패치까지 — secret leak 추적

# 2. regex 로 (-S 와 다름)
git log -G 'foo\s*=\s*\d+'           # 매치되는 줄이 추가/제거된 commit

# 3. 현재 코드 안에서 검색 — grep 보다 빠름
git grep 'FOO_BAR'                   # tracked 파일만 대상
git grep -n 'FOO_BAR'                # 라인 번호
git grep -l 'FOO_BAR'                # 파일명만
git grep -i 'foo_bar'                # 대소문자 무시
git grep -E 'foo|bar'                # extended regex
git grep -c 'TODO'                   # 파일별 매치 수

# 4. 특정 시점의 코드 검색
git grep 'FOO_BAR' v1.2.0
git grep 'FOO_BAR' HEAD~5

# 5. commit 메시지 검색
git log --grep='hotfix'
git log --grep='bug' --grep='fix' --all-match    # 두 단어 동시
git log --grep='ABC-123' --all      # 다른 브랜치까지

# 6. 저자 검색
git log --author='masterway'
git log --author='@gmail.com'

# 7. 파일명 / 경로
git log --diff-filter=A -- src/api.py   # 파일 추가된 시점
git log --diff-filter=D -- src/api.py   # 삭제된 시점
git log --follow -- src/api.py          # rename 따라가기

# 8. branch 들 안에서 검색
git grep 'FOO' $(git rev-list --all)    # 모든 commit (느림)
git log --all -S 'FOO'                  # 모든 브랜치
```

## 사용법
- 보안 사고 = `git log -S 'API_KEY' --all -p` 로 노출 commit 추적
- 함수 origin = `git log -S 'def my_func' -- src/api.py`
- 빠른 grep = `git grep`

## 주의
- `-S` 는 "등장 횟수 변화" → 같은 줄 안의 수정은 안 잡힘 → `-G` 사용
- `git grep` 은 untracked 파일 제외 — 새 파일은 `add -N` 또는 일반 `grep`
- 큰 repo 의 `--all -p` 는 시간 / 메모리 ↑

## 관련
- [[snippet-git-log]] — log 자체의 다른 필터
- [[snippet-git-blame]] — 한 줄의 출처
