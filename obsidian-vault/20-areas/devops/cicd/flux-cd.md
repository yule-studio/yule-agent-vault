---
title: "Flux CD — GitOps CD (lightweight)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:34:00+09:00
tags: [devops, cicd, flux, gitops]
---

# Flux CD — GitOps CD (lightweight)

**[[cicd|↑ cicd]]**

---

## 1. 무엇

- ArgoCD 와 같은 GitOps but UI 없이 CLI / CRD 중심.
- Kustomize / Helm release CRD.
- multi-tenancy 강.

---

## 2. 설치

```bash
brew install fluxcd/tap/flux
flux bootstrap github \
    --owner=myorg \
    --repository=fleet-infra \
    --branch=main \
    --path=clusters/prod
```

---

## 3. GitRepository + Kustomization

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata: {name: app, namespace: flux-system}
spec:
  url: https://github.com/myorg/app
  ref: {branch: main}
  interval: 1m

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: {name: app, namespace: flux-system}
spec:
  interval: 5m
  path: ./deploy
  prune: true
  sourceRef: {kind: GitRepository, name: app}
  targetNamespace: production
```

---

## 4. HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: {name: web, namespace: production}
spec:
  interval: 5m
  chart:
    spec:
      chart: ./helm
      sourceRef: {kind: GitRepository, name: app}
  values:
    replicaCount: 3
```

---

## 5. Flux vs ArgoCD

| | Flux | ArgoCD |
| --- | --- | --- |
| UI | X (CLI) | 강 |
| 학습 | 약간 ↑ | 쉬움 |
| multi-tenancy | 강 | 가능 |
| 인기 | 2위 | 1위 |

→ 본 vault: ArgoCD (UI 선호) 시작 → 대규모 multi-tenant 시 Flux 검토.

---

## 관련

- [[cicd|↑ cicd]]
- [[argocd-cd]]
- [[../kubernetes/gitops-argocd]]
