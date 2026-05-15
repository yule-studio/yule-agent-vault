---
title: "GitOps with ArgoCD / Flux ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:58:00+09:00
tags: [devops, kubernetes, gitops, argocd, flux]
---

# GitOps with ArgoCD / Flux ★

**[[kubernetes|↑ k8s]]**

---

## 1. GitOps 란

- **git = source of truth** (manifest / Helm values).
- **CD = git sync** (push → cluster 자동 reconcile).
- 변경 = PR + review (audit).
- rollback = git revert.

---

## 2. ArgoCD vs Flux

| | **ArgoCD** ★ | Flux |
| --- | --- | --- |
| UI | 강 (대시보드) | 없음 (CLI) |
| 학습 | 쉬움 | 약간 ↑ |
| multi-cluster | 강 | 강 |
| 진화 | active | active |
| 인기 | 1위 | 2위 |

→ 본 vault: **ArgoCD**.

---

## 3. ArgoCD 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d

# UI 접근
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## 4. Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
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
    syncOptions:
      - CreateNamespace=true
```

---

## 5. 디렉토리 구조 (제 권장)

```
k8s-manifests/
├── base/                      # Kustomize 또는 Helm chart
│   └── web/
│       ├── deployment.yaml
│       └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    └── production/
```

---

## 6. App-of-Apps 패턴 (대규모)

```yaml
# argocd 안의 모든 Application 을 관리하는 Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: apps, namespace: argocd}
spec:
  source:
    repoURL: ...
    path: apps/        # 안에 다른 Application 들의 YAML
```

→ ArgoCD UI 1 클릭으로 모든 환경 / 앱 동기.

---

## 7. 흐름

```
1. dev push code
2. CI (GitHub Actions) → image build + push registry
3. CI → k8s-manifests repo 의 image tag 갱신 (PR + auto merge or manual)
4. ArgoCD 가 git change 감지 → cluster sync
5. health check (자동 rollback 옵션)
```

---

## 8. 함정

1. **drift** — 누가 cluster 직접 변경 시 ArgoCD 가 reset (selfHeal=true 시).
2. **secret git commit** — sealed-secrets / SOPS / External Secrets Operator.
3. **CRD 누락** → Application sync 실패.
4. **자동 prune** + 실수 manifest 삭제 → 리소스 삭제.
5. **single repo** + 여러 환경 — 잘못 merge 시 prod 영향.

---

## 9. 관련

- [[kubernetes|↑ k8s]]
- [[helm]]
- [[../cicd/cicd|↗ cicd]]
- [[../iac/iac|↗ iac]]
