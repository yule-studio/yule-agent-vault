---
title: "ConfigMaps + Secrets"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:48:00+09:00
tags: [devops, kubernetes, configmap, secret]
---

# ConfigMaps + Secrets

**[[kubernetes|↑ k8s]]**

---

## 1. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata: {name: app-config}
data:
  log.level: INFO
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://pg:5432/db
```

```yaml
# env
env:
  - name: LOG_LEVEL
    valueFrom: {configMapKeyRef: {name: app-config, key: log.level}}

# volume mount
volumes:
  - name: config
    configMap: {name: app-config}
containers:
  - volumeMounts:
      - name: config
        mountPath: /app/config
```

---

## 2. Secret (base64, NOT encrypted)

```yaml
apiVersion: v1
kind: Secret
metadata: {name: app-secret}
type: Opaque
stringData:
  DB_PASSWORD: supersecret    # k8s 가 base64 자동
```

⚠️ base64 ≠ 암호화. `etcd encryption at rest` + Vault / sealed-secrets 권장.

---

## 3. Secret 안전한 관리

| 방법 | 비교 |
| --- | --- |
| **sealed-secrets** (Bitnami) ★ | 공개 git 에 commit 가능 |
| **External Secrets Operator** | Vault / AWS SM / GCP SM 동기 |
| **HashiCorp Vault** | 강력 + 운영 부담 |
| **AWS Secrets Manager / GCP SM** | cloud 통합 |
| **SOPS** (Mozilla) | GitOps 친화 (KMS encrypt) |

---

## 4. 사용 패턴

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: {name: app-secret, key: DB_PASSWORD}

# 또는 file mount
volumes:
  - name: secret
    secret: {secretName: app-secret}
```

---

## 5. 환경별 (overlay)

```
base/
├── deployment.yaml
└── configmap.yaml
overlays/
├── dev/configmap-patch.yaml
└── prod/configmap-patch.yaml
```

→ Kustomize 또는 Helm values.

---

## 6. 함정

1. **secret git commit** → 영구 leak.
2. **env 변경 후 pod restart 안 함** → 옛 값 유지.
3. **secret 평문 etcd** — KMS encryption at rest 활성화.
4. **모든 Pod 에 같은 secret read 권한** → RBAC 으로 제한.
5. **configmap size 1MB+** → split / external storage.

---

## 7. 관련

- [[kubernetes|↑ k8s]]
- [[rbac]]
- [[../security-ops/security-ops|↗ security-ops]]
