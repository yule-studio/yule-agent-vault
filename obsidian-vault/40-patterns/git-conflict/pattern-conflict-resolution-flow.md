---
title: "Conflict Resolution Flow — 충돌 해결 단계별 흐름"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:15:00+09:00
tags:
  - pattern
  - git
  - conflict
  - workflow
---

# Pattern: Conflict Resolution Flow

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[git-conflict|↑ git-conflict]]**

---

## 의도
충돌이 났을 때 **즉흥적으로 마커만 지우는 것** 이 아니라, 정해진 단계 (preflight → backup → attempt → resolve → verify) 를 따라간다.

---

## 동기

충돌 해결 사고의 90% 는:

1. 의도 파악 안 한 채 `<<<<<<<` 지우기 → 한쪽 의도 사라짐
2. resolve 후 컴파일만 확인 → 런타임 / 테스트 실패
3. `git checkout --ours / --theirs` 잘못 해석 (merge / rebase 에서 의미 반대)
4. 충돌 파일 외 영역도 같이 변경 (`-X theirs` 의 부작용)

→ **고정된 단계** 가 즉흥성을 제거한다.

---

## 단계 다이어그램

```
[0] preflight    : 충돌의 규모 / 영역 파악
       ↓
[1] backup       : backup/<source>-<date> push
       ↓
[2] attempt      : merge / rebase 시도 → 충돌 표면화
       ↓
[3] understand   : 각 충돌마다 양쪽 의도 읽기
       ↓
[4] resolve      : 한 파일씩 해결 + add
       ↓
[5] verify       : marker / test / diff 검증
       ↓
[6] commit       : 의미 있는 메시지로 commit
       ↓
[7] propagate    : push / PR
```

---

## 적용 예

### [0] Preflight — 규모 파악

```bash
# 어떤 파일이 충돌할지 미리 보기 (실제 merge 없이)
git fetch origin
git merge-tree $(git merge-base HEAD origin/main) HEAD origin/main | head -100

# 또는 dry-run 식 (실제 머지 후 abort)
git merge --no-commit --no-ff origin/main
git status                           # 충돌 파일 목록
git merge --abort                    # 되돌리기
```

→ "충돌 5개 / 큰 파일 1개" 인지 / "100개 폭주" 인지 먼저 확인. 후자면 [[pattern-large-conflict-split]] 로.

---

### [1] Backup — [[pattern-backup-branch]] 그대로

```bash
git branch backup/feature-x-$(date +%Y%m%d-%H%M)
git push origin backup/feature-x-$(date +%Y%m%d-%H%M)
```

---

### [2] Attempt — 실제 시도

```bash
# 머지 시도용 브랜치에서 (또는 직접 feature/x 에서)
git switch -c resolve/feature-x-conflicts feature/x
git merge origin/main
# ... CONFLICT (content): src/api.py, src/db.py ...
git status
# Unmerged paths:
#   both modified: src/api.py
#   both modified: src/db.py
```

---

### [3] Understand — 양쪽 의도 읽기 (★ 가장 중요)

```bash
# 충돌 파일을 IDE 로 열어서:
# <<<<<<< HEAD                    ← 내 (resolve 브랜치 기준)
# our code
# ||||||| merged common ancestors ← 공통 조상 (zdiff3 켜야 보임)
# original code
# =======
# their code                       ← 머지하려는 쪽 (origin/main)
# >>>>>>> origin/main
```

각 충돌마다:
- **why ours** — 내가 왜 이렇게 바꿨나 (작업 PR / commit 메시지로)
- **why theirs** — 상대가 왜 이렇게 바꿨나 (git log -p 로)
- **resolution** — 둘 다 / 한쪽 / 새로 합친 형태

```bash
# 그 줄을 누가 / 왜 만들었나
git log --merge -- src/api.py            # 두 쪽의 commit 만
git log -p --merge -- src/api.py         # diff 까지
git blame src/api.py                     # 라인별
```

---

### [4] Resolve — 한 파일씩

```bash
# 4-1. IDE 에서 직접 편집 (가장 자주)
code src/api.py
# <<<<<<< / ======= / >>>>>>> 삭제 + 의도 보존하는 코드

# 4-2. 한쪽 통째로 (의미 명확할 때만)
git checkout --ours src/legacy_only.py     # 내 변경 유지 (merge 기준)
git checkout --theirs src/auto_generated.py # 상대 변경 유지

# 주의: rebase 중에는 ours/theirs 의 의미 반대 (HEAD = 머지하려는 쪽)

# 4-3. 그래픽 도구
git mergetool                            # 미리 설정한 도구 (meld / vimdiff)

# 4-4. 한 파일 해결 끝 → add
git add src/api.py
```

---

### [5] Verify — 검증 (★ 빠뜨리기 쉬움)

```bash
# 5-1. 충돌 marker 잔존 검사
git diff --check                         # whitespace + marker
grep -RnE '^(<{7}|={7}|>{7}) ' .         # 직접 grep
# string literal 안에 marker 있으면 위 둘 다 못 잡음 → 빌드/테스트로

# 5-2. 빌드 / 테스트
npm test     # or pytest, ./gradlew test, etc.
npm run lint
npm run type-check

# 5-3. 의도 검증
git diff feature/x..HEAD -- src/api.py   # 내 작업 대비 추가된 것이 합리적인가
git diff origin/main..HEAD -- src/api.py # main 대비 변경이 작업의 의도와 맞나
```

---

### [6] Commit — 좋은 메시지

```bash
git commit -m "merge: resolve feature/x vs main conflicts

변경 이유:
- origin/main 의 db schema 변경과 feature/x 의 api 변경이 같은 파일 (src/db.py) 수정

주요 변경 사항:
- src/api.py: ours (feature 의 라우팅 유지)
- src/db.py: theirs 의 schema + ours 의 ORM 매핑 양쪽 적용
- src/legacy.py: theirs 그대로 (legacy 정리는 main 의 일관성)

비고:
- backup: backup/feature-x-20260515
- 검증: pytest + npm run type-check 둘 다 green
"
```

→ commit 메시지에 **어떤 파일이 어떤 의도로 해결됐는지** 남긴다. 미래의 git blame 이 고마워한다.

---

### [7] Propagate

```bash
git push -u origin resolve/feature-x-conflicts
gh pr create --base feature/x --head resolve/feature-x-conflicts \
  --label conflict-resolution --assignee codwithyc
```

---

## 트레이드오프

### 장점
- 순서가 정해져서 패닉 모드에서도 따라갈 수 있음
- "verify 안 하고 push" 사고 차단
- commit 메시지 의 형태가 표준화 → 미래 디버깅 ↑

### 단점
- 작은 충돌엔 과한 절차 (1-2 파일 / 30 초면 stage [3]~[5] 합쳐도 됨)
- IDE 의 충돌 해결 UI 가 좋으면 [3] 의 일부를 자동화

---

## 안티패턴

- **marker 만 지우기** — 양쪽 의도 안 읽음
- **`-X theirs` 일괄** — 의미 잃음
- **테스트 안 돌리고 push** — runtime 실패
- **commit 메시지 "fix conflicts"** — 무슨 충돌이었는지 영원히 모름
- **rebase 중 매번 `--skip`** — 누락된 commit / 변경 추적 불가

---

## 관련

- [[git-conflict|↑ git-conflict]]
- [[pattern-backup-branch]]
- [[pattern-merge-resolution-split]]
- [[pattern-large-conflict-split]]
- [[../../50-snippets/git/snippet-git-merge-rebase|↗ merge/rebase 명령어]]
