---
title: "Pipeline 표준 패턴 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:36:00+09:00
tags: [devops, cicd, pattern]
---

# Pipeline 표준 패턴 ★

**[[cicd|↑ cicd]]**

---

## 1. 표준 7 단계

```
1. checkout
2. dependency cache restore
3. lint + test (unit + integration)
4. build (compile / image)
5. scan (SAST + image CVE + secret detect)
6. push (artifact / image)
7. deploy (env / GitOps manifest update)
```

---

## 2. branch-based strategy

| Branch / Tag | 동작 |
| --- | --- |
| `feature/*` PR | test only |
| `main` push | test + build + scan + push → staging deploy |
| `tag v*` | + production deploy |

---

## 3. parallel + dependency

```yaml
# GitHub Actions 의 needs
jobs:
  test-unit:
  test-integration:
    needs: [test-unit]
  build:
    needs: [test-unit, test-integration]
  scan:
    needs: [build]
  deploy:
    needs: [scan]
```

→ test 둘 다 PASS 후 build.

---

## 4. fail-fast

- 빠른 cancel — 1 job fail → 나머지 cancel.
- 가능 한 빠른 단계에서 (lint > test > build).

---

## 5. progressive promote

```
PR → preview env
  ↓ merge
main → staging env (auto)
  ↓ manual approve
production env (canary 10% → 50% → 100%)
```

---

## 6. rollback 자동화

```yaml
# k8s — rollout 실패 시 자동
spec:
  progressDeadlineSeconds: 300
  strategy:
    rollingUpdate:
      maxUnavailable: 0
```

```bash
# 또는 ArgoCD pre/post-sync hook
kubectl rollout undo deploy/web
```

---

## 7. artifact / image tag 전략

- `<semver>` + `<git-sha>` 함께.
- `latest` X.
- registry retention policy (옛 tag 자동 정리).

---

## 8. notification

```yaml
# Slack
- uses: rtCamp/action-slack-notify@v2
  if: failure()
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
    SLACK_MESSAGE: "❌ CI failed on ${{ github.ref }}"
```

---

## 9. 함정

1. **모든 단계 직렬** → 느림 → 병렬.
2. **scan 안 함** → CVE 누출.
3. **manual deploy 만** → release 늦음.
4. **rollback 자동화 X** → 실패 시 사람 개입 늦음.
5. **secret 의 echo / log** → 누설.

---

## 10. 관련

- [[cicd|↑ cicd]]
- [[github-actions]]
- [[secret-in-pipeline]]
- [[release-strategies]]
