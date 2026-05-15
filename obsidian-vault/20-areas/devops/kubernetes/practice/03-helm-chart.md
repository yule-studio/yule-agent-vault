---
title: "실습 03 — Helm chart 작성"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:10:00+09:00
tags: [devops, kubernetes, practice, helm]
---

# 실습 03 — Helm chart 작성

**[[practice|↑ practice]]**

> 1시간 — 02 실습의 manifest 를 Helm chart 로.

---

## 1. chart 생성

```bash
helm create mychart
cd mychart
```

→ 기본 template (deployment + service + ingress + HPA + ServiceAccount).

---

## 2. values.yaml 수정

```yaml
replicaCount: 2

image:
  repository: demo
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: demo.local
      paths:
        - path: /
          pathType: Prefix

resources:
  limits: {cpu: 500m, memory: 512Mi}
  requests: {cpu: 100m, memory: 256Mi}

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: prod
```

---

## 3. 환경별

```yaml
# values-dev.yaml
replicaCount: 1
ingress:
  hosts:
    - host: dev.demo.local

# values-prod.yaml
replicaCount: 5
autoscaling:
  maxReplicas: 30
```

---

## 4. 설치

```bash
helm install demo ./mychart -f values-dev.yaml
helm upgrade demo ./mychart -f values-prod.yaml
helm list
helm history demo
helm rollback demo 1
```

---

## 5. Subchart (PostgreSQL 의존)

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "16.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: pg
```

```bash
helm dependency update
helm install demo ./mychart \
    --set pg.auth.password=demopw
```

---

## 6. lint / dry-run

```bash
helm lint ./mychart
helm template demo ./mychart > out.yaml      # render manifest
helm install demo ./mychart --dry-run --debug
```

---

## 7. Package + push (chart museum / OCI)

```bash
helm package ./mychart                       # mychart-0.1.0.tgz
helm push mychart-0.1.0.tgz oci://ghcr.io/myorg/charts
```

---

## 8. 다음

- ArgoCD 로 자동 sync → [[../gitops-argocd]]
- CI/CD 통합 → [[../../cicd/cicd]]
