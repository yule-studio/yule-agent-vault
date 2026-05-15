---
title: "Conflict Prevention — 충돌 예방"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:25:00+09:00
tags:
  - pattern
  - git
  - conflict
  - prevention
---

# Pattern: Conflict Prevention

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[git-conflict|↑ git-conflict]]**

---

## 의도
충돌이 났을 때의 해결도 중요하지만, **애초에 충돌이 작거나 안 나게** 만드는 게 더 싸다.

---

## 동기

충돌의 원인 = 다음의 곱:
```
충돌 = (브랜치 수명) × (변경 영역 중첩) × (소통 부재)
```

각 변수를 줄이면 충돌 자체가 사라지거나 작아진다.

---

## 7 가지 예방 원칙

### 1. Small + Short-lived PR
- PR 의 수명 ≤ 2-3 일
- diff ≤ 400 줄 (research: 400+ 가 되면 리뷰 품질 급락)
- 큰 feature → 작은 PR 시리즈로 분할 (`feat: add X (1/3)` ...)

### 2. 자주 main 동기화
```bash
# 작업 시작 / 끝 / 점심 후 등 자주
git fetch origin
git rebase origin/main      # 또는 git merge --ff-only origin/main
```
→ **하루 한 번 이상** 권장. 1 주일 묵힌 브랜치 = 폭탄.

### 3. 작업 영역 분리 (소통)
- 같은 파일 / 함수를 동시에 만지는 팀원 있는지 PR 만들기 전에 확인
- Slack / Issue 의 "이 파일 만지는 중" comment
- 큰 리팩터링은 **공지 후 1 commit 에 끝내고 즉시 머지** (long-lived 금지)

### 4. 포맷팅 commit 분리
- prettier / black / gofmt 같은 일괄 포맷팅 = **별도 commit / 별도 PR**
- `.git-blame-ignore-revs` 에 등록 → blame 에 안 잡힘
- 일반 작업과 섞으면 모든 PR 에 포맷팅 충돌이 동행

### 5. rerere 활성
```bash
git config --global rerere.enabled true
git config --global rerere.autoUpdate true
```
→ 같은 충돌을 한 번 해결하면 다음 번 자동 적용. long-lived 브랜치 / 자주 rebase 하는 패턴에서 큰 효과.

### 6. merge.conflictstyle = zdiff3
```bash
git config --global merge.conflictstyle zdiff3
```
→ 충돌 표기에 **공통 조상** 까지 표시. "어디서부터 갈라졌나" 보여서 의도 파악 ↑.

```
<<<<<<< HEAD
ours
||||||| merged common ancestors    ← 이게 추가됨
original
=======
theirs
>>>>>>> branch
```

### 7. 큰 파일 / 자동 생성 파일 처리
- `package-lock.json` / `yarn.lock` / `Gemfile.lock` 등 = **자동 생성** → 한쪽 채택 + 재생성
  ```bash
  git checkout --theirs package-lock.json
  npm install                    # 재생성
  git add package-lock.json
  ```
- migration / schema 파일 = **타임스탬프 충돌** → 둘 다 채택 + 번호 / 시간 재정렬
- snapshot 테스트 / generated TS = 별도 처리 룰

---

## 도구 / 자동화

### CI 의 자동 검사
```yaml
# .github/workflows/conflict-check.yml
on:
  pull_request:
    branches: [main]
jobs:
  conflict-check:
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: |
          git fetch origin main
          # PR base 와 dry-run merge
          if ! git merge-tree $(git merge-base origin/main HEAD) HEAD origin/main | grep -q '<<<<<<<'; then
            echo "no conflict"
          else
            echo "::warning::PR has conflicts with main — rebase 권장"
            exit 1
          fi
```

→ PR 머지 가능성 매번 검사.

### 브랜치 보호 룰
- **"branches are up-to-date" 강제** → 머지 전에 main rebase 강제
- **squash merge only** → long-lived 가 사실상 차단
- **stale branch auto-delete** → 1주 inactive 자동 삭제 알림

---

## 트레이드오프

### 장점
- 충돌 빈도 ↓ + 충돌 크기 ↓
- 리뷰 품질 ↑ (작은 PR)
- blame / history 가 명확

### 단점
- 작은 PR = PR 개수 ↑ → 리뷰 부담은 분산 (총합은 비슷)
- 자주 rebase = 강제 rebase 부담 + force-push 분쟁 (resolved by `--force-with-lease`)
- 팀 합의 / 문화 변화 필요

---

## 안티패턴

- **"기능 다 끝나면 한번에 머지"** — long-lived = 충돌 폭탄
- **포맷팅 / lint fix 를 일반 commit 에 섞기** — 모든 PR 에 형식 충돌
- **`rerere` 끄고 같은 충돌 반복** — 시간 + 의도 일관성 둘 다 망가짐
- **lock 파일을 사람이 손으로 머지** — 양쪽 dependency 가 어긋남

---

## 체크리스트

PR 만들기 전:
- [ ] PR diff ≤ 400 줄?
- [ ] base 가 origin/main 최신?
- [ ] 같은 파일 만지는 동료 확인?
- [ ] 포맷팅 변경 분리?
- [ ] lock 파일 재생성 / 검증?

---

## 관련

- [[git-conflict|↑ git-conflict]]
- [[pattern-large-conflict-split]] — 예방 실패 시 대응
- [[../github-conventions/pattern-pr-template]] — PR 사이즈 정책
- [[../../50-snippets/git/snippet-git-config-alias|↗ rerere / zdiff3 설정]]
