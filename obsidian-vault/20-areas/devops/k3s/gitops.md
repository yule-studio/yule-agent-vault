---
title: "k3s GitOps — ArgoCD / Fleet"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:15:00+09:00
tags: [devops, k3s, gitops, argocd, fleet]
---

# k3s GitOps — ArgoCD / Fleet

**[[k3s|↑ k3s]]**

---

## 1. 왜

```
edge / 여러 cluster — manual kubectl 불가능:
  - 매장 100개 cluster 에 같은 manifest 적용
  - app 변경 시 git push → 자동
  - drift 자동 fix

→ k3s + GitOps = 표준.
```

---

## 2. 선택지

| | 무엇 |
| --- | --- |
| **ArgoCD** | k8s 표준 GitOps, UI 친화 |
| **Fleet** (Rancher) | 매우 많은 cluster (수천) — k3s 친화 |
| **FluxCD** | code-first, Helm 친화 |
| **Rancher (Manager)** | UI, multi-cluster, RBAC |

→ **소규모 = ArgoCD**, **수십~수천 cluster = Fleet / Rancher**.

---

## 3. ArgoCD 설치 (k3s 위)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# UI 접속
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d
```

또는 Helm:
```bash
helm install argocd argo/argo-cd -n argocd --create-namespace
```

---

## 4. ArgoCD Application

```yaml
# 1. Git repo 에 manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: my-app, namespace: argocd}
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app
    targetRevision: main
    path: k8s/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true                # git 에서 삭제하면 cluster 에서도
      selfHeal: true             # drift 자동 fix
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff: {duration: 5s, factor: 2, maxDuration: 3m}
```

→ git push → ArgoCD 가 polling (default 3분) 또는 webhook → 즉시 sync.

---

## 5. App-of-Apps 패턴

```yaml
# 한 Application 이 다른 Application 들을 정의
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: {name: all-apps, namespace: argocd}
spec:
  source:
    repoURL: https://github.com/myorg/cluster
    path: apps/
  destination: {server: https://kubernetes.default.svc, namespace: argocd}
  syncPolicy: {automated: {selfHeal: true}}
```

```
git repo 구조:
  cluster/
    apps/
      monitoring.yaml    ← Application (prometheus)
      ingress.yaml       ← Application (nginx-ingress)
      cert-manager.yaml  ← Application
```

→ "한 git repo = 한 cluster". cluster 추가 = 새 git repo.

---

## 6. ApplicationSet (★ 여러 cluster)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: {name: monitoring, namespace: argocd}
spec:
  generators:
    - clusters: {}                    # 모든 등록된 cluster
  template:
    metadata: {name: 'monitoring-{{name}}'}
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/clusters
        path: monitoring
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy: {automated: {selfHeal: true}}
```

→ 새 cluster 등록 시 자동 monitoring stack 배포.

---

## 7. Fleet (★ 여러 k3s cluster)

```bash
# Rancher 가 control. 또는 standalone:
helm install fleet-crd fleet/fleet-crd -n cattle-fleet-system
helm install fleet fleet/fleet -n cattle-fleet-system
```

```yaml
# GitRepo
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata: {name: my-app, namespace: fleet-default}
spec:
  repo: https://github.com/myorg/my-app
  branch: main
  paths: [manifests]
  targets:
    - name: prod
      clusterSelector:
        matchLabels: {env: prod}
    - name: staging
      clusterSelector:
        matchLabels: {env: staging}
```

→ cluster 마다 label → target 자동.

---

## 8. k3s manifests/ 자동 배포 (★ 옵션)

```yaml
# /var/lib/rancher/k3s/server/manifests/ 폴더의 yaml 자동 apply
# 예: nginx-ingress 미리 배포
```

```bash
sudo cp my-app.yaml /var/lib/rancher/k3s/server/manifests/
# k3s 가 자동 apply (5초 polling)
```

→ kustomize / helm 도 가능 (HelmChart CRD).

→ **GitOps 미적용 시** 이걸로 임시 가능. 하지만 GitOps 권장.

---

## 9. Helm Chart 자동 배포

```yaml
# /var/lib/rancher/k3s/server/manifests/my-chart.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata: {name: nginx-ingress, namespace: kube-system}
spec:
  repo: https://kubernetes.github.io/ingress-nginx
  chart: ingress-nginx
  version: 4.10.0
  targetNamespace: ingress-nginx
  valuesContent: |-
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
```

→ k3s 가 자동 helm install / upgrade.

---

## 10. 흔한 흐름 (★)

```
1. cluster bootstrap (config.yaml + ArgoCD install)
2. ArgoCD root application (App-of-Apps)
3. 각 component (cert-manager / monitoring / ingress 등) Application
4. application 들 ApplicationSet 으로 환경별
5. dev → staging → prod (branch / overlay)
6. PR 으로 변경 → review → merge → 자동 deploy
7. ArgoCD UI 에서 drift / sync status 확인
```

---

## 11. secrets

```
git 에 secret 그대로 X.

방법:
  A. SealedSecret (Bitnami)
  B. SOPS + age (kustomize plugin)
  C. External Secrets Operator + Vault / AWS SM
  D. ArgoCD CMP (Custom Management Plugin)
```

→ [[../security-ops/secrets-management|secrets-management]] 참조.

---

## 12. progressive delivery (Argo Rollouts)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: my-app}
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - {setWeight: 20}
        - {pause: {duration: 10m}}
        - {setWeight: 50}
        - {pause: {duration: 10m}}
        - {setWeight: 100}
      analysis:
        templates:
          - {templateName: success-rate}
        startingStep: 2
```

→ canary + 자동 검증 (Prometheus query)+ rollback.

---

## 13. 함정

1. **manifests/ 폴더 + GitOps 동시 사용** — drift / 충돌.
2. **automated: selfHeal: true 만 — git delete = cluster 즉시 삭제** ★ 위험. prune 별도.
3. **ArgoCD Application name 충돌** — namespace 분리.
4. **webhook 안 설정** — 3분 polling 만 (느림).
5. **secret in git 평문** — SealedSecret / SOPS / ESO.
6. **rollback git revert** — git 만 revert, image 는 옛 tag 필요.
7. **ApplicationSet generator 잘못** — 모든 cluster 에 잘못된 manifest.

---

## 14. 관련

- [[k3s|↑ k3s]]
- [[../cicd/argocd-cd|↗ ArgoCD]]
- [[../cicd/flux-cd|↗ FluxCD]]
- [[../platform-engineering/self-service|↗ self-service GitOps]]
