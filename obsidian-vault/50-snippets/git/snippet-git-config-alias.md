---
title: "git config / alias — 매일 쓰는 설정"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:30:00+09:00
tags:
  - snippet
  - git
  - config
  - alias
---

# git config / alias — 매일 쓰는 설정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
한번 설정하면 매일 reuse — 동료에게 추천해도 좋은 합리적 기본값 + 짧은 alias.

## 코드

```bash
# === 사용자 / repo 별 정체성
git config --global user.name "Your Name"
git config --global user.email "you@x.com"
git config user.email "work@x.com"          # 이 repo 만 회사 이메일

# === 자동완성 / 색
git config --global color.ui auto
git config --global core.editor "code --wait"
git config --global init.defaultBranch main

# === pull / push 안전
git config --global pull.ff only           # ff 안 되면 거부 (명시적 rebase/merge)
git config --global push.default current   # 현재 브랜치 → 동명 원격
git config --global push.autoSetupRemote true   # -u 자동
git config --global fetch.prune true       # fetch 시 사라진 원격 정리

# === rebase / merge 향상
git config --global rebase.autoStash true  # rebase 시 자동 stash
git config --global rebase.autoSquash true # fixup! 자동 정렬
git config --global merge.conflictstyle zdiff3   # 충돌 표기에 origin 도

# === 보존 / 안전
git config --global rerere.enabled true    # 같은 충돌 자동 해결
git config --global core.autocrlf input    # macOS/Linux
git config --global core.autocrlf true     # Windows
git config --global help.autocorrect 20    # 오타 자동 교정 (2초 대기)

# === alias ★
git config --global alias.st 'status -sb'
git config --global alias.co 'checkout'
git config --global alias.sw 'switch'
git config --global alias.br 'branch'
git config --global alias.cm 'commit -m'
git config --global alias.amend 'commit --amend --no-edit'
git config --global alias.unstage 'restore --staged'
git config --global alias.undo 'reset HEAD~1 --mixed'
git config --global alias.lg "log --graph --pretty=format:'%C(yellow)%h %C(cyan)%an %C(green)%ar%Creset %s %C(auto)%d'"
git config --global alias.last 'log -1 HEAD --stat'
git config --global alias.aliases 'config --get-regexp ^alias\\.'

# === blame 의 포맷팅 commit 무시
echo "abc123 # prettier 적용" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs

# === credential
git config --global credential.helper osxkeychain   # macOS
git config --global credential.helper cache         # 메모리 임시
git config --global credential.helper 'cache --timeout=3600'

# === 현재 설정 보기
git config --list --global
git config --list --local
git config --get user.email
```

## 사용법
- 새 머신 셋업 = 이 파일 그대로 복붙 → `~/.gitconfig` 동기화
- 회사 / 개인 분리 = `~/.gitconfig` 에 `[includeIf "gitdir:~/work/"] path=~/.gitconfig-work`

## 주의
- `--global` vs `--local` vs `--system` 우선순위 (local > global > system)
- alias 안의 shell 명령 = `!` 접두사 (`alias.fix '!git add -p && git commit --amend --no-edit'`)
- `core.autocrlf` 잘못 설정 = 전체 파일이 변경된 것처럼 보임

## 관련
- [[snippet-git-log]] — `lg` alias 활용
- [[snippet-git-remote]] — push / pull 의 안전 옵션
