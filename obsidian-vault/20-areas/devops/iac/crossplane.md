---
title: "Crossplane — k8s native IaC"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:16:00+09:00
tags: [devops, iac, crossplane, kubernetes]
---

# Crossplane — k8s native IaC

**[[iac|↑ iac]]**

---

## 1. 무엇

- k8s CRD 로 cloud 자원 정의.
- ArgoCD + Crossplane = "manifest YAML 하나로 cloud + app".
- composition 으로 자체 abstraction (XR / XRD).

---

## 2. 설치

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
    -n crossplane-system --create-namespace
```

```bash
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata: {name: provider-aws}
spec:
  package: xpkg.upbound.io/upbound/provider-aws-rds:v1
EOF
```

---

## 3. AWS RDS 예시

```yaml
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: prod-db
spec:
  forProvider:
    region: ap-northeast-2
    instanceClass: db.t3.medium
    engine: postgres
    engineVersion: "16"
    allocatedStorage: 20
    username: admin
    password:
      secretKeyRef: {name: db-secret, namespace: default, key: password}
  writeConnectionSecretToRef:
    name: prod-db-conn
    namespace: default
```

```bash
kubectl apply -f rds.yaml
kubectl get instance.rds.aws.upbound.io
```

---

## 4. Composition (자체 추상)

```yaml
# CompositeResourceDefinition + Composition 으로 자체 Kind
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: {name: xpostgresqls.acme.com}
spec:
  group: acme.com
  names: {kind: XPostgreSQL, plural: xpostgresqls}
  claimNames: {kind: PostgreSQL, plural: postgresqls}
  versions: [...]
```

→ "PostgreSQL" 라는 자체 자원 → 내부적으로 RDS + Subnet + SG 자동.

---

## 5. Crossplane vs Terraform

| | Crossplane | Terraform |
| --- | --- | --- |
| 위치 | k8s 안 | 외부 CLI |
| GitOps | ArgoCD 와 native | TF 별도 |
| state | etcd | external |
| 추상 | XRD / Composition | module |
| 본 vault | k8s 위주 시 | 다른 | 표준 |

---

## 6. 관련

- [[iac|↑ iac]]
- [[tools-comparison]]
- [[../kubernetes/gitops-argocd|↗ ArgoCD]]
