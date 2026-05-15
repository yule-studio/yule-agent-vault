---
title: "ArgoCD — GitOps CD"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:32:00+09:00
tags: [devops, cicd, argocd, gitops]
---

# ArgoCD — GitOps CD

**[[cicd|↑ cicd]]**

> 자세히: [[../kubernetes/gitops-argocd]].

---

## 1. 빠른 시작

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 2. Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: web, namespace: argocd}
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: main
    path: production/web
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 3. Argo Rollouts (canary)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: web}
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
```

자세히: [[../kubernetes/deployments]].

---

## 관련

- [[cicd|↑ cicd]]
- [[../kubernetes/gitops-argocd]]
- [[flux-cd]]
