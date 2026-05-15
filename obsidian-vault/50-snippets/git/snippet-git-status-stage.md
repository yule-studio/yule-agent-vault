---
title: "git status / add — 상태 + 스테이징"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:15:00+09:00
tags:
  - snippet
  - git
  - status
  - staging
---

# git status / add — 상태 + 스테이징

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
지금 어떤 상태인지 + 어디까지 스테이지로 올릴지를 정확히 통제한다.

## 코드

```bash
# 1. 상태 보기
git status
git status -s                  # short — `M file.py / ?? new.py`
git status -sb                 # short + branch 정보
git status -uall               # untracked 파일 전부 보기 (큰 디렉토리)

# 2. 스테이지에 추가
git add file.py                # 한 파일
git add src/                   # 디렉토리
git add -u                     # tracked 의 변경만 (새 파일 무시)
git add -A                     # 모든 변경 (untracked 포함)

# 3. interactive — 일부만 스테이지 (★ 가장 강력)
git add -p file.py             # patch 모드 — hunk 별 y/n 선택
# 한 파일 안에서 commit A / commit B 로 나눠 담을 때 필수

# 4. 스테이지 취소 (변경은 유지)
git restore --staged file.py   # 신문법 (Git 2.23+)
git reset HEAD file.py         # 구문법

# 5. 변경 자체를 되돌림 (working dir 복원, 변경 사라짐)
git restore file.py
git checkout -- file.py        # 구문법

# 6. untracked 파일 보기 / 무시
git status --ignored           # .gitignore 에 무시된 것도 표시

# 7. 안 보이는 변경 (CRLF / mode / encoding)
git diff --raw                 # 모드 비트 변경 확인
git config core.fileMode false # mode 비트 무시 (macOS ↔ Linux)

# 8. 새 파일을 commit 없이 tracked 로
git add --intent-to-add new.py # 또는 `git add -N`
```

## 사용법
- 코드 review 전: `git add -p` 로 의미 단위로 분리 → 작은 commit 여러 개
- "이거 commit 하려고 한 거 맞아?" = `git diff --staged` 로 항상 한번 더 확인

## 주의
- `git add .` 와 `git add -A` 둘 다 비밀파일 (`.env` / credentials) 까지 담을 수 있음 → 명시적 add 권장
- `git restore` 는 **되돌릴 수 없다** — 작업 손실 주의

## 관련
- [[snippet-git-diff]] — 스테이지 전후 diff
- [[snippet-git-undo]] — 더 큰 되돌리기
