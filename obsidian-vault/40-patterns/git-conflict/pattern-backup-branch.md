---
title: "Backup Branch — 위험한 머지/rebase 전 백업 브랜치"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:05:00+09:00
tags:
  - pattern
  - git
  - conflict
  - backup
---

# Pattern: Backup Branch

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[git-conflict|↑ git-conflict]]**

---

## 의도
위험한 git 작업 (rebase / 큰 머지 / `reset --hard` 직전) 에서 **현재 상태를 그대로 보존** 한다.
실패 시 reflog 가 아니라 명시적 브랜치로 복구한다.

---

## 동기 (Motivation)

- `reflog` 는 로컬에만 — 다른 머신 / 새 clone 에서는 못 봄
- reflog 의 유효기간 = 30 일 (`gc.reflogExpireUnreachable`)
- `--force` push 후 깨달은 실수 = 원격에 백업 없으면 복구 어려움
- 백업 브랜치 만들기 = 1 초, push 1 초. **비용이 0 에 가깝다**

---

## 구조

```
원본 branch                    백업 branch (이름 컨벤션)
  feature/x          ──→       backup/feature/x-20260515
                                backup/feature/x-pre-rebase
                                backup/feature/x-pre-merge-main

  main               ──→       backup/main-20260515-before-revert
```

이름 패턴:
```
backup/<원본>-<날짜>
backup/<원본>-pre-<위험한-작업>
```

→ 폴더형 prefix (`backup/`) 로 일반 작업 브랜치와 시각적 분리 + 일괄 정리 쉬움.

---

## 적용 예

### 5.1 기본 — 로컬 + 원격 백업

```bash
# 위험 작업 직전, 작업 브랜치에서
git switch feature/x
git branch backup/feature/x-$(date +%Y%m%d)
git push origin backup/feature/x-$(date +%Y%m%d)

# 이제 안심하고 위험 작업
git rebase main         # 충돌 폭주
# 망했다 → 백업으로 복구
git switch backup/feature/x-$(date +%Y%m%d)
git branch -D feature/x
git switch -c feature/x
```

### 5.2 alias 로 자동화

```bash
git config --global alias.bk \
  '!f() { d=$(date +%Y%m%d-%H%M); b=$(git symbolic-ref --short HEAD); git branch backup/${b}-${d} && git push origin backup/${b}-${d}; }; f'

# 사용
git bk
# → backup/feature/x-20260515-1330 자동 생성 + push
```

### 5.3 pre-rebase 자동 백업 (hook)

`.git/hooks/pre-rebase`:
```bash
#!/bin/bash
b=$(git symbolic-ref --short HEAD 2>/dev/null) || exit 0
git branch "backup/${b}-pre-rebase-$(date +%s)" 2>/dev/null
```

→ rebase 누를 때마다 자동 백업.

---

## 트레이드오프

### 장점
- 복구 = `git switch <backup>` 한 줄
- 원격 push 한 백업은 머신 분실에도 안전
- 동료가 "내 작업 어디 갔어요?" 할 때 직접 보여줄 수 있음

### 단점
- 백업 브랜치가 누적되면 `git branch -a` 가 지저분
- 원격 백업 push = 약간의 트래픽 / 저장공간

### 정리 정책
```bash
# 30 일 지난 백업 일괄 삭제
git for-each-ref --format='%(refname:short) %(committerdate:short)' refs/heads/backup/ |
  awk -v cutoff="$(date -v-30d +%Y-%m-%d)" '$2 < cutoff { print $1 }' |
  xargs -I{} git branch -D {}

# 원격도
git push origin --delete <branch>
```

→ 분기별 정리. 또는 작업 끝나고 main 머지된 백업만 삭제.

---

## 안티패턴

- **백업을 commit message 로만 남김** ("WIP backup") — 위 commit 하나만 잡힐 뿐, 브랜치 자체 보존 X
- **`stash` 로 대신** — stash 도 로컬 / 영구성 X / 다른 머신 못 봄
- **백업 한 번 만들고 영원히 안 지움** — repo 가 백업 천국이 됨 (정리 정책 필수)
- **원격 push 안 함** — 머신 고장 / 디스크 손상에 무력

---

## 언제 안 만들어도 되는가
- 작업이 1 commit 이고 push 안 한 상태 + reflog 충분 (24시간 안)
- `git rebase --abort` 가 명백히 동작할 수 있는 단순 rebase

→ 그래도 1초밖에 안 걸리므로 **만드는 게 기본**.

---

## 관련

- [[git-conflict|↑ git-conflict]]
- [[pattern-merge-resolution-split]] — 백업 후 머지 시도
- [[../../50-snippets/git/snippet-git-reflog|↗ reflog]] — 백업 안 만들었을 때의 안전망
- [[../../50-snippets/git/snippet-git-config-alias|↗ alias 만들기]]
