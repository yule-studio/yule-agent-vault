---
title: "Helm — k8s 패키지 매니저 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:50:00+09:00
tags: [devops, kubernetes, helm, package]
---

# Helm — k8s 패키지 매니저 ★

**[[kubernetes|↑ k8s]]**

---

## 1. 무엇

- k8s 의 npm / apt — chart 단위로 다중 리소스 묶음.
- 환경별 (dev / staging / prod) values 만 다르게.
- rollback 1 명령.

---

## 2. Chart 구조

```
mychart/
├── Chart.yaml              # 메타
├── values.yaml             # default values
├── templates/
│   ├── deployment.yaml     # Go template
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   └── _helpers.tpl        # 공통
└── charts/                 # sub-charts (의존)
```

---

## 3. Template 예시

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: 1.0
resources:
  limits: {cpu: 500m, memory: 512Mi}
  requests: {cpu: 100m, memory: 256Mi}
```

---

## 4. 환경별 values

```bash
helm install web ./mychart \
  -f values.yaml \
  -f values.prod.yaml \
  --set replicaCount=5
```

```yaml
# values.prod.yaml
replicaCount: 10
ingress:
  hosts: [api.example.com]
```

---

## 5. 자주 쓰는 명령

```bash
helm install web ./mychart -n prod
helm upgrade web ./mychart -n prod -f values.prod.yaml
helm history web -n prod
helm rollback web 2 -n prod          # 이전 release 로
helm list -A
helm template web ./mychart > out.yaml   # dry-run rendering
helm lint ./mychart
helm package ./mychart                # tgz
```

---

## 6. Helm vs Kustomize

| | **Helm** ★ | Kustomize |
| --- | --- | --- |
| 패키지 | O (Chart.tgz) | X |
| 템플릿 | Go template | overlay (patch) |
| 의존성 | sub-chart | base + overlay |
| 학습 곡선 | 약간 ↑ | 단순 |
| 본 vault | 외부 chart (Bitnami / cert-manager) | 자체 앱 (옵션) |

---

## 7. 유명 charts

- bitnami/postgresql
- bitnami/redis
- ingress-nginx/ingress-nginx
- jetstack/cert-manager
- prometheus-community/kube-prometheus-stack
- grafana/loki-stack
- argo/argo-cd

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install pg bitnami/postgresql -n db
```

---

## 8. 함정

1. **chart 만 수정 + values 안 분리** → 환경 별 관리 어려움.
2. **`helm upgrade --install`** 사용 권장 (install / upgrade 통합).
3. **release name 충돌** — namespace 별 unique.
4. **CRD 는 chart 안 X** — 사전 install.
5. **secret values commit** → SOPS / sealed-secrets.

---

## 9. 관련

- [[kubernetes|↑ k8s]]
- [[gitops-argocd]]
- [[../cicd/cicd|↗ cicd]]
