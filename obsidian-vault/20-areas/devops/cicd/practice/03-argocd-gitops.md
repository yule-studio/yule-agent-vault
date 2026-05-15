---
title: "실습 03 — GitOps with ArgoCD"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:54:00+09:00
tags: [devops, cicd, practice, argocd, gitops]
---

# 실습 03 — GitOps with ArgoCD

**[[practice|↑ practice]]**

> 1시간 — push-based 에서 pull-based 로.

---

## 1. ArgoCD 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
# admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d
```

---

## 2. manifest repo 분리

```
my-org/
├── app-source/                     # 코드 + Dockerfile + CI workflow
└── app-manifests/                  # k8s YAML / Helm (ArgoCD 가 watch)
    ├── production/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── staging/
```

---

## 3. CI workflow update — manifest repo bump

```yaml
# app-source/.github/workflows/ci.yml — image push 후
  - name: Bump manifest tag
    if: github.ref == 'refs/heads/main'
    env:
      MANIFEST_TOKEN: ${{ secrets.MANIFEST_REPO_PAT }}
    run: |
      git clone https://x:$MANIFEST_TOKEN@github.com/my-org/app-manifests.git
      cd app-manifests
      sed -i "s|image: ghcr.io/my-org/app:.*|image: ghcr.io/my-org/app:sha-${GITHUB_SHA::7}|" production/deployment.yaml
      git config user.email "ci@example.com"
      git config user.name "CI"
      git commit -am "bump prod to sha-${GITHUB_SHA::7}"
      git push
```

---

## 4. Application

```yaml
# manifest repo 에 또는 ArgoCD UI 에서 생성
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: web, namespace: argocd}
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/app-manifests
    targetRevision: main
    path: production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 5. 흐름 확인

```
1. dev push code → app-source main
2. GitHub Actions: build + push image
3. GitHub Actions: app-manifests 의 image tag bump (commit)
4. ArgoCD (3분 polling) 감지 → cluster sync
5. UI 에서 "Synced + Healthy"
```

---

## 6. rollback

```bash
# manifest repo 의 옛 commit 으로 revert
git revert HEAD
git push
# ArgoCD 자동 sync
```

또는 ArgoCD UI 의 "History" → "Rollback".

---

## 7. 다음

- Argo Rollouts (canary) 추가 → [[../../kubernetes/deployments]]
- multi-cluster ArgoCD → [[../argocd-cd]]
