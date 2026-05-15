---
title: "Trunk-based development — small batch / fast CI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:30:00+09:00
tags: [devops, cicd, trunk-based]
---

# Trunk-based development — small batch / fast CI

**[[cicd|↑ cicd]]**

---

## 1. 무엇

```
main (trunk) branch 가 하나.
개발자가 짧은 branch 만 (수시간 - 1-2일).
빠르게 merge 후 main 에 통합.

vs Git Flow:
  Git Flow: long-lived develop / release branch
  Trunk:    main 만 + short topic branch
```

→ DORA 의 elite 회사의 공통점.

---

## 2. 흐름

```
1. main 에서 short branch 생성
2. 작은 변경 (수시간)
3. PR open + CI 자동
4. review + merge
5. main 자동 deploy → staging → prod
6. feature flag 로 release 분리

→ 매일 N번 PR / merge.
```

---

## 3. branch 정책

```
short-lived:
  feature/JIRA-123-add-button   (1-2일)
  hotfix/fix-timeout            (1시간)
  
NOT:
  develop                       (X)
  release/v1.0                  (X)
  long-lived feature branch    (X)

원칙:
  - main 이 항상 deployable
  - branch 1-2일 maximum
  - merge 빈번 (rebase / merge commit)
```

---

## 4. PR 정책

```
small PR (★):
  - 200 lines 이하 권장
  - 500 lines + = 분리

review SLA:
  - same-day review
  - 4 hour 안에 첫 응답

merge:
  - squash merge (clean history)
  - 또는 rebase + merge
  - merge commit X (noisy)
```

→ "큰 PR" = code review 망침.

---

## 5. feature flag (★ 필수)

```
deploy ≠ release:
  feature 완성 → main merge → 배포 → flag off
  → 사용자에게 안 보임
  → 안전하게 점진 ramp
```

```yaml
# LaunchDarkly / Flagsmith / Unleash 등
if (featureFlag.isEnabled("new-checkout", user)) {
    return newCheckout();
} else {
    return oldCheckout();
}
```

→ "코드 merge" 와 "feature release" 분리.

---

## 6. branch protection

```
GitHub branch protection (main):
  ☐ require PR (push 안 됨)
  ☐ require CI pass
  ☐ require reviewer approval (1+)
  ☐ require linear history (rebase)
  ☐ no force push
  ☐ no delete
  ☐ require signed commit
  ☐ require status check (test / lint / scan)
```

---

## 7. CI 의 속도 (★)

```
trunk-based 의 핵심 = CI 빠름.

목표:
  CI 시간 < 10분 (★)
  PR feedback < 15분

방법:
  - parallel test
  - test selection (changed code 만)
  - cache (deps, image)
  - 작은 docker image
  - merge queue (Bors / GitHub queue)
```

---

## 8. continuous integration vs delivery vs deployment

```
CI (Continuous Integration):
  - main 으로 자주 merge + 자동 test
  
CD (Continuous Delivery):
  - merge → staging 자동 deploy
  - prod 는 manual approve
  
CD (Continuous Deployment):
  - merge → prod 자동 deploy (no manual)

trunk-based + CD = elite team.
```

---

## 9. Git Flow 와 비교

```
Git Flow:
  develop ← feature
  release ← develop
  main ← release
  hotfix ← main
  → 복잡, long-lived branch, merge hell

Trunk-based:
  main ← short topic branch
  → 단순, 빠른 통합

언제 Git Flow:
  - mobile app (App Store version)
  - 큰 release cycle
  - legacy team
  
대부분 web / SaaS = trunk-based.
```

---

## 10. dark launch / canary (★)

```
새 feature 의 ramp:

Stage 1: 코드 merge + flag off
Stage 2: internal user 만 (1%)
Stage 3: beta users (10%)
Stage 4: 점진 ramp (10% → 50% → 100%)
Stage 5: flag 제거 + 옛 코드 cleanup

→ 작은 영향 + 큰 안전성.
```

---

## 11. monorepo + trunk-based

```
monorepo (Google / Meta):
  모든 service 가 한 repo.
  trunk-based 의 자연.
  
도구:
  - Nx (JS/TS)
  - Turborepo
  - Bazel (large scale)
  - Lerna
  - pnpm workspaces

incremental build:
  - 변경된 module 만 build / test
  - dependency graph
```

→ team / service 별 polyrepo 도 trunk-based 가능.

---

## 12. test pyramid

```
1. unit test         많이, fast (ms)
2. integration test  중간 (s)
3. contract test     service 간
4. e2e test          적게, slow (m)

→ unit / integration 80%, e2e 20%.
```

---

## 13. branch by abstraction (★ 큰 refactor)

```
큰 refactor = long-lived branch ❌
→ "branch by abstraction":

1. 새 abstraction layer 도입 (interface)
2. 기존 코드 → interface 통해 사용
3. 새 구현 점진 도입
4. flag 으로 switch
5. 옛 구현 제거

→ main 에서 계속 변경 + ship.
```

---

## 14. 변화 관리

```
"Git Flow → Trunk-based" 마이그:
  1. CI 빠르게 (가장 큰 lever)
  2. feature flag 도입
  3. PR size 작게
  4. develop branch 폐기 (어색)
  5. main protection 강화
  6. 점진 (한 service 부터)
  7. DORA metric 측정
```

→ tool / process 보다 culture 가 어려움.

---

## 15. 함정

1. **trunk + slow CI** — bottleneck (merge queue 폭주).
2. **flag 없이 trunk** — broken code 가 main 에.
3. **review 없이 merge** — quality ↓.
4. **PR 너무 큼** — review 의미 X.
5. **flag 영구 누적** — tech debt.
6. **main 의 unstable** — 즉시 revert.
7. **mobile / 큰 release cycle** 에 강제 trunk — 부적합.

---

## 16. 관련

- [[cicd|↑ cicd]]
- [[feature-flags-integration]]
- [[dora-metrics]]
- [[../platform-engineering/scaffolding|↗ scaffolding]]
