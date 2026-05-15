---
title: "Merge / Resolution Branch Split — 머지용 / 해결용 분리"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:10:00+09:00
tags:
  - pattern
  - git
  - conflict
  - branch-strategy
---

# Pattern: Merge / Resolution Branch Split

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[git-conflict|↑ git-conflict]]**

---

## 의도
충돌을 다룰 때 **작업 브랜치 자체에서 해결하지 않고**, 3 개의 분리된 브랜치로 단계화한다:

1. **backup/...** — 원상 복구 안전망
2. **merge/<feature>-into-<base>** — 머지 시도 (충돌 dry-run)
3. **resolve/<feature>-conflicts** — 충돌 해결만 담는 브랜치 (별도 PR / 리뷰 가능)

---

## 동기 (Motivation)

작업 브랜치에서 바로 머지하면 생기는 문제:

- 작업 history 에 "충돌 해결" commit 이 섞여서 PR 리뷰가 흐려짐
- 잘못 해결했을 때 작업 commit 까지 되돌려야 함
- "충돌 해결의 의도가 맞나" 를 **다른 사람이 리뷰할 단위가 없음** (작업 PR 안에 묻힘)
- 해결 자체가 큰 작업 (수십 파일) 이면 PR 사이즈가 폭증

해결: 머지 시도 + 충돌 해결 = **독립된 1차 산출물**.

---

## 구조

```
main ─────●─────────●─────────●──── (계속 진행)
                              ↑ [4] resolve PR merged
feature/x ───●───●───●        │
              ↑   ↑   ↑        │
   [1] backup/feature/x─20260515 (안전망)
   [2] merge/feature/x-into-main ←(시도) 충돌!
                              │
   [3] resolve/feature/x-conflicts ←(별도 PR — 리뷰)
```

### 브랜치 명명
```
backup/<source>-<date>            # 패턴 1
merge/<source>-into-<target>      # 머지 시도
resolve/<source>-conflicts        # 충돌 해결
resolve/<source>-vs-<target>      # 양쪽 명시 (대안)
```

---

## 적용 예

### 5.1 단계별 명령

```bash
# === [0] 사전 정보 수집
git switch feature/x
git fetch origin
git log --oneline origin/main..HEAD     # 내 작업 commit
git log --oneline HEAD..origin/main     # base 가 얼마나 앞서 있나
git merge-base feature/x origin/main    # 공통 조상

# === [1] 백업 브랜치 (패턴 1)
git branch backup/feature/x-$(date +%Y%m%d)
git push origin backup/feature/x-$(date +%Y%m%d)

# === [2] 머지 시도 브랜치
git switch -c merge/feature-x-into-main feature/x
git merge origin/main
# 충돌이 났으면:
#   Auto-merging src/api.py
#   CONFLICT (content): Merge conflict in src/api.py
# 충돌 상태에서 다음 단계로 이동 (resolve 브랜치로 옮김)

# === [3] 충돌 해결 브랜치
# 충돌 상태 그대로 새 브랜치로 옮기는 트릭
git switch -c resolve/feature-x-conflicts          # 충돌 그대로 가져옴
# 또는 깔끔히 처음부터:
git merge --abort
git switch -c resolve/feature-x-conflicts feature/x
git merge origin/main

# 이제 충돌 해결 (snippet-git-conflict-resolution-flow 참고)
git status
# edit src/api.py — <<<<<<< / ======= / >>>>>>> 정리
git add src/api.py
git commit -m "fix: resolve conflicts feature/x vs main (api.py 의 X 충돌)"

# === [4] 해결 결과 검증
git diff feature/x..HEAD --stat         # 무엇을 추가로 더 했나
git diff origin/main..HEAD --stat       # main 대비 최종 변경

# === [5] PR
git push -u origin resolve/feature-x-conflicts
gh pr create \
  --base feature/x \
  --head resolve/feature-x-conflicts \
  --title "resolve: feature/x vs main conflicts" \
  --label conflict-resolution \
  --assignee codwithyc

# → 머지되면 feature/x 가 origin/main 의 변경을 다 받은 상태
# → 이제 feature/x → main 으로 일반 PR (충돌 없이 머지)
```

### 5.2 누가 무엇을 리뷰

| PR | 리뷰 포인트 | 누구 |
| --- | --- | --- |
| `resolve/* → feature/x` | 충돌 해결의 의도 (원작자 변경이 살아있나) | feature 작성자 |
| `feature/x → main` | 기능 자체 | 평소 리뷰어 |

→ 두 PR 의 리뷰 관심사가 다르다 — 분리가 자연스러움.

---

## 트레이드오프

### 장점
- 충돌 해결 = **별도 commit / 별도 PR** — history 깔끔
- 잘못 해결했으면 resolve PR 만 close → 작업 브랜치 영향 0
- 큰 충돌일 때 여러 commit 으로 분할 가능 (resolve 브랜치 안에서 단계별)
- 원작자가 "내 의도 살아있나" 만 보면 됨 (작업 PR 검토와 분리)

### 단점
- 브랜치 / PR 수 증가 — 작은 충돌엔 과함
- 팀 모두에게 흐름 공유 필요
- merge / resolve 브랜치를 main 에 push 하려면 PR 정책 / CI 가 받아줘야 함

### 언제 쓰면 안 되는가
- 충돌 = 1-2 파일 + 1 분이면 해결되는 단순 변경
- 혼자 일하는 사이드 프로젝트
- hotfix — 시간 압박이 ↑ 인 경우 (단순 머지 + 백업이면 충분)

---

## 안티패턴

- **resolve 브랜치를 직접 main 으로 머지** — base 가 잘못됨. resolve 는 feature 브랜치에 머지, feature 가 main 으로 가야 함
- **merge 시도 브랜치에서 충돌 해결까지 다 함** — 두 의도가 섞여서 history 추적 어려움
- **resolve PR 에 기능 변경도 끼워 넣음** — "충돌 풀다가 버그 보여서 고침" 금지. 별도 PR
- **`-X theirs` 로 일괄 해결 + resolve PR** — 의미 있는 리뷰 불가능

---

## 관련

- [[git-conflict|↑ git-conflict]]
- [[pattern-backup-branch]]
- [[pattern-conflict-resolution-flow]] — resolve 브랜치 안에서의 해결 절차
- [[pattern-large-conflict-split]] — resolve 자체를 여러 PR 로 분할
- [[../github-conventions/pattern-workflow]] — PR 흐름 안의 위치
