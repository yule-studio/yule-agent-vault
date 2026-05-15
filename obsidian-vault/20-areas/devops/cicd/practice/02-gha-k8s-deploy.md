---
title: "실습 02 — GitHub Actions → k8s 자동 배포"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:52:00+09:00
tags: [devops, cicd, practice, kubernetes]
---

# 실습 02 — GitHub Actions → k8s 자동 배포

**[[practice|↑ practice]]**

> 1시간 — image push 후 k8s deployment 자동 update.

---

## 1. kubeconfig secret 등록

```bash
# kubeconfig 의 일부만 (deploy SA 의 권한 제한)
cat ~/.kube/config | base64
```

→ GitHub Settings → Secrets → `KUBE_CONFIG_B64`.

또는 더 안전 — IRSA / OIDC (cloud).

---

## 2. workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: production    # GitHub UI 의 environment + approval
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with: {version: 'v1.30.0'}

      - name: Decode kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG_B64 }}" | base64 -d > ~/.kube/config

      - name: Deploy
        run: |
          IMAGE=ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          kubectl set image deploy/web app=$IMAGE -n production
          kubectl rollout status deploy/web -n production --timeout=5m

      - name: Smoke test
        run: |
          kubectl run smoke-${{ github.run_id }} \
            --image=curlimages/curl:latest \
            --rm --restart=Never -i -- \
            curl -f http://web.production/actuator/health
```

---

## 3. environment + approval (수동 승인 옵션)

```
Settings → Environments → Production
  Required reviewers: [admin]
  Wait timer: 5 minutes
  Deployment branches: main
```

→ deploy job 이 approval 대기.

---

## 4. rollback workflow

```yaml
# .github/workflows/rollback.yml
on: workflow_dispatch

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - run: kubectl rollout undo deploy/web -n production
```

→ GitHub Actions UI 에서 "Run workflow" 클릭.

---

## 5. 다음

[[03-argocd-gitops]] — pull-based GitOps 로 전환.
