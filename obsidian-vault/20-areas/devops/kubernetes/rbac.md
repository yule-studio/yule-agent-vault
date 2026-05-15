---
title: "RBAC — Role / RoleBinding / ServiceAccount"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:56:00+09:00
tags: [devops, kubernetes, rbac, security]
---

# RBAC — Role / RoleBinding / ServiceAccount

**[[kubernetes|↑ k8s]]**

---

## 1. 4 개념

| | 무엇 |
| --- | --- |
| **ServiceAccount** | Pod 이 cluster API 호출 시 identity |
| **Role / ClusterRole** | 허용 작업 정의 (namespace / cluster) |
| **RoleBinding / ClusterRoleBinding** | SA / User / Group ↔ Role |
| **User / Group** | 사람 (cluster 외부 인증) |

---

## 2. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
```

```yaml
spec:
  template:
    spec:
      serviceAccountName: my-app
      containers: [...]
```

---

## 3. Role + RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: pod-reader-bind, namespace: production}
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. ClusterRole (cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: node-reader}
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

→ ClusterRoleBinding 으로 모든 namespace.

---

## 5. AWS IAM Role for ServiceAccount (IRSA) — EKS

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-app-role
```

→ Pod 가 AWS API 호출 시 IAM Role 사용 (AK / SK 불필요).

---

## 6. GCP Workload Identity (GKE)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    iam.gke.io/gcp-service-account: my-app@project.iam.gserviceaccount.com
```

---

## 7. 함정

1. **`cluster-admin`** → 모든 권한 (PROD 에서 사용 X).
2. **default SA + 권한** → 다른 Pod 도 영향.
3. **resource name `"*"`** → wildcard 위험.
4. **AK/SK 평문 secret** → IRSA / Workload Identity 권장.
5. **RoleBinding 의 namespace 누락** → 동작 X.

---

## 8. 관련

- [[kubernetes|↑ k8s]]
- [[configmaps-secrets]]
- [[../security-ops/security-ops|↗ security-ops]]
