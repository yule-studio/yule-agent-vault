---
title: "git-conflict — 충돌 대비 / 해결 패턴 hub"
kind: knowledge
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:00:00+09:00
tags:
  - pattern
  - git
  - conflict
  - workflow
  - hub
---

# git-conflict — 충돌 대비 / 해결 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 정리 |

**[[../patterns|↑ 40-patterns]]**

> 머지 / rebase 충돌은 **사고가 아니라 빈번한 정상 이벤트**.
> 충돌이 났을 때 당황하지 않도록 사전에 **브랜치 위생** + **단계별 해결 흐름** 을 패턴화한다.

---

## 1. 큰 그림

```
┌─ 작업 브랜치 (feature/x)
│
├─ [1] backup/feature/x-20260515         ← 패턴 1: 무조건 백업
│
├─ [2] merge/feature/x-into-main         ← 패턴 2: 머지 시도용
│       └ 충돌 발견
│
├─ [3] resolve/feature/x-conflicts       ← 패턴 2: 해결 작업용
│       └ 충돌만 해결 → PR (별도 리뷰)
│
└─ [4] 다시 feature/x 로 통합 → main 머지
```

→ 한 브랜치에서 다 하지 않는 게 핵심. 각 단계가 독립적으로 폐기 / 재시도 가능.

---

## 2. 언제 어떤 패턴

| 상황 | 패턴 |
| --- | --- |
| 충돌이 날 것 같다 / 위험한 머지 | [[pattern-backup-branch]] |
| 충돌이 났고, 해결을 별도 PR 로 분리하고 싶다 | [[pattern-merge-resolution-split]] |
| 충돌 해결 자체의 단계가 헷갈린다 | [[pattern-conflict-resolution-flow]] |
| 충돌이 파일 100+ / 너무 크다 | [[pattern-large-conflict-split]] |
| 매번 같은 충돌이 반복된다 / 자주 다시 충돌난다 | [[pattern-conflict-prevention]] |

---

## 3. "백업 → 머지용 → 해결용" 의 이유 (3 분 요약)

### 3.1 백업 브랜치
- `reset --hard` / 잘못된 `rebase --abort` 후 reflog 도 30 일 한도
- **원격 push 한 백업** 만 진짜 안전
- 비용 거의 0 — 무조건 만든다

### 3.2 머지용 브랜치 (merge attempt)
- 작업 브랜치 (feature/x) 자체에서 머지 시도하면 **충돌 흔적이 작업 history 에 남음**
- merge 만 하는 임시 브랜치로 분리하면, 충돌 보고 → 폐기 → 재시도가 자유로움
- "이 머지가 실제로 안전한가?" 의 dry-run

### 3.3 해결용 브랜치 (conflict resolution)
- 충돌 해결 = 사실상 **두 사람의 변경을 한 사람이 결정** 하는 위험 작업
- 별도 브랜치 + 별도 PR 로 만들면:
  - 원작자가 "내 의도가 살아있는지" 리뷰 가능
  - CI 가 해결 결과만 따로 빌드 / 테스트
  - 잘못 해결했으면 PR drop 만 하면 됨

---

## 4. 안티패턴

- **로컬에서 `git merge` 하고 충돌 해결 후 바로 push** — 백업 없음, 검증 없음
- **`-X theirs` / `-X ours` 로 일괄 해결** — 의도 모르고 한쪽 변경 전부 버림
- **`rebase` 중 충돌 → `--skip` 남발** — 누락된 변경 모름
- **충돌 파일의 `<<<<<<<` 마커가 commit 에 그대로** — 컴파일은 되는데 런타임 망가짐 (어떻게? — 마커가 string literal 안에 있으면 grep 도 못 잡음)

---

## 5. 도구

| 도구 | 의미 |
| --- | --- |
| `git rerere` | 같은 충돌 자동 해결 (한 번 해결한 것 기억) |
| `git mergetool` | 비주얼 머지 도구 (vimdiff / meld / VS Code) |
| `git checkout --conflict=diff3` | 충돌 표기에 공통 조상 포함 (zdiff3 권장) |
| `git diff --check` | 충돌 마커 남았는지 검사 |

설정:
```bash
git config --global rerere.enabled true
git config --global merge.conflictstyle zdiff3
```

---

## 6. 관련

- [[../patterns|↑ 40-patterns]]
- [[../github-conventions/pattern-workflow]] — PR 흐름 안에서의 충돌
- [[../../50-snippets/git/git|↗ git snippets]]
- [[../../50-snippets/git/snippet-git-merge-rebase|↗ merge/rebase]]
- [[../../50-snippets/git/snippet-git-reflog|↗ reflog 복구]]
