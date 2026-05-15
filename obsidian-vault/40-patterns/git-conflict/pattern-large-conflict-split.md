---
title: "Large Conflict Split — 큰 충돌 분할"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:20:00+09:00
tags:
  - pattern
  - git
  - conflict
  - large-merge
---

# Pattern: Large Conflict Split

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[git-conflict|↑ git-conflict]]**

---

## 의도
충돌이 **수십~수백 파일 / 여러 도메인** 에 걸쳐 있을 때, 한 번에 해결하지 않고 **부분 머지 / 부분 해결** 로 쪼갠다.

---

## 동기

큰 충돌의 흔한 패턴:
- 한 달 떨어진 long-lived feature 브랜치를 main 에 머지
- 대규모 리네임 / 포맷팅 commit 이 base 에 들어옴
- 두 큰 리팩터링이 동시에 진행됐다가 만남

한 번에 해결하면:
- PR 의 diff 수천 줄 — 리뷰 불가능
- 충돌 한 곳 잘못 풀면 모두 다시
- 도중 멈추기 어려움 (충돌 상태로 working 묶임)

→ **분할 정복** + 각 단계가 main 으로 머지 가능한 작은 단위.

---

## 구조 — 3 전략

```
[A] 도메인별 분할
    main + auth 변경 만 → PR1
    main + db 변경 만 → PR2
    main + api 변경 만 → PR3

[B] 시간순 분할 (incremental rebase)
    main 의 commit 을 한 묶음씩 끌어와서 해결
    feature ← main[1..10]   해결 → PR1
    feature ← main[11..20]  해결 → PR2

[C] Strangler Fig 흉내
    main 으로 새 코드 (작업의 일부) 미리 이식
    → feature 의 충돌 면적 자체를 줄임
```

---

## 적용 예

### A. 도메인별 분할

```bash
# feature/x 가 auth + db + api + ui 4 도메인을 건드림
# main 과의 충돌 = 40 파일

# A-1: auth 만 먼저
git switch -c resolve/feature-x-auth feature/x
git checkout origin/main -- '!(auth)/*' 'src/auth/*'  # auth 만 feature 변경 유지
# 또는 명시적으로:
git switch -c resolve/feature-x-auth main
git checkout feature/x -- src/auth/

git commit -m "merge: feature/x auth changes into main"
git push -u origin resolve/feature-x-auth
# → PR → main 으로

# A-2: db, A-3: api, A-4: ui ... 차례로 main 에 들어감
# 매번 feature/x 를 main 기준으로 rebase 하면 충돌 면적 ↓
```

### B. 시간순 분할 — incremental rebase

```bash
# main 의 commit 30개가 쌓여있을 때
git switch feature/x
git branch backup/feature-x-pre-rebase

# 10개씩 끌어와서 rebase
main_commits=$(git log --oneline HEAD..origin/main | wc -l)
chunk=10

# 1차: 가장 옛 10개
git rebase $(git rev-parse origin/main~$((main_commits-chunk)))
# 충돌 해결 → continue → push 백업
git push origin feature/x:backup/feature-x-after-chunk1

# 2차: 다음 10개
git rebase $(git rev-parse origin/main~$((main_commits-chunk*2)))
# ...

# n차: 최종 main
git rebase origin/main
```

→ 매 chunk 마다 검증 / 백업. 중간에 멈출 수 있음.

### C. Strangler Fig — 충돌 면적 줄이기

```bash
# feature/x 의 일부 (호환되는 새 코드) 를 main 에 미리 이식
git switch -c precursor/new-helper-from-feature-x main
git checkout feature/x -- src/utils/new_helper.py
git commit -m "feat: add new_helper (will be used by feature/x)"
# → PR → main 머지

# feature/x 는 이제 main 의 new_helper 를 쓰는 형태로 rebase
git switch feature/x
git rebase main
# new_helper 부분의 충돌 = 사라짐
```

→ feature/x 의 자기 충돌 압력 자체를 미리 분산.

---

## 어느 전략을 언제

| 상황 | 전략 |
| --- | --- |
| 충돌이 **파일 / 디렉토리** 별로 명확히 분리 | A (도메인별) |
| 충돌이 **시간 / commit 흐름** 으로 누적 | B (시간순) |
| 작업이 long-lived + 미리 합칠 부분이 식별됨 | C (Strangler) |
| 한 파일 안의 큰 충돌 | A 안 → 같은 파일을 hunk 별로 분할 commit |

---

## 도구

```bash
# 충돌 파일 목록 + 크기
git diff --name-only --diff-filter=U
git diff --check                     # marker 잔존
git diff --stat origin/main..HEAD    # 충돌 영역 stat

# 한 hunk 만 해결 / 적용
git checkout --patch origin/main src/api.py   # patch 모드

# rerere 가 같은 패턴 반복 해결 자동 (필수)
git config --global rerere.enabled true
```

---

## 트레이드오프

### 장점
- 각 PR 이 작아져 리뷰 가능
- 중간 단계가 main 에 들어가서 다른 사람도 활용 가능
- 한 단계 잘못 풀어도 그 단계만 재시도

### 단점
- 전체 시간은 더 걸림 (분할의 오버헤드)
- 도메인 의존성이 강하면 분할 불가능한 영역 존재
- 팀 합의 / 일정 조율 필요 (PR 들이 며칠~몇 주 분산)

---

## 안티패턴

- **충돌이 큰데 한 PR 로 밀어붙임** — 리뷰 없음 / 실수 광범위
- **분할 안 했는데 분할한 척** — 한 PR 안에 도메인 섞임
- **rerere 안 쓰고 같은 충돌 반복** — 시간 / 의도 일관성 둘 다 망가짐
- **분할의 중간 단계가 broken** — main 이 망가짐. 각 PR 이 자체로 green 이어야 함

---

## 관련

- [[git-conflict|↑ git-conflict]]
- [[pattern-merge-resolution-split]]
- [[pattern-conflict-resolution-flow]]
- [[pattern-conflict-prevention]] — 애초에 큰 충돌이 안 나게
- [[../refactoring-patterns/pattern-strangler-fig|↗ Strangler Fig 패턴]]
