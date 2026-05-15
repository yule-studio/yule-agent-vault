---
title: "Tekton — k8s native CI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:30:00+09:00
tags: [devops, cicd, tekton, kubernetes]
---

# Tekton — k8s native CI

**[[cicd|↑ cicd]]**

---

## 1. 무엇

- k8s CRD 로 pipeline 정의.
- Task / Pipeline / PipelineRun.
- ArgoCD + Tekton = full GitOps CI/CD.
- 학습 어렵 (DSL 없음, YAML 만).

---

## 2. 설치

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

---

## 3. Task (재사용 단위)

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata: {name: build-image}
spec:
  params:
    - name: image
      type: string
  workspaces:
    - name: source
  steps:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - --dockerfile=Dockerfile
        - --context=$(workspaces.source.path)
        - --destination=$(params.image)
```

---

## 4. Pipeline (Task 조합)

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata: {name: build-push-deploy}
spec:
  params:
    - {name: image, type: string}
  workspaces:
    - {name: shared-data}
  tasks:
    - name: fetch
      taskRef: {name: git-clone}
      workspaces:
        - {name: output, workspace: shared-data}
    - name: test
      taskRef: {name: gradle-test}
      runAfter: [fetch]
      workspaces:
        - {name: source, workspace: shared-data}
    - name: build
      taskRef: {name: build-image}
      runAfter: [test]
      params: [{name: image, value: $(params.image)}]
```

---

## 5. PipelineRun (실행)

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata: {generateName: build-}
spec:
  pipelineRef: {name: build-push-deploy}
  params:
    - {name: image, value: ghcr.io/myorg/app:1.0}
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources: {requests: {storage: 5Gi}}
```

---

## 6. Tekton trigger (webhook)

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata: {name: github-push}
spec:
  interceptors:
    - ref: {name: github}
  bindings: [...]
  template:
    spec:
      ...
```

---

## 7. Tekton + ArgoCD + 카프카 같은 GitOps Stack

```
git push
  ↓
Tekton (build + scan + push)
  ↓
GitOps manifest repo 갱신
  ↓
ArgoCD sync
  ↓
k8s deploy
```

---

## 8. 함정

1. **YAML 가 매우 verbose** — Helm / Kustomize 로 wrap.
2. **DSL 없음** — Jenkins groovy 처럼 직접 변수 / loop 제한.
3. **debug 어려움** — Pod logs.
4. **운영 부담** — Tekton 자체도 관리.

---

## 9. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
- [[argocd-cd]]
- [[../kubernetes/kubernetes|↗ k8s]]
