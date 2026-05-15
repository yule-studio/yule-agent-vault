---
title: "Crossplane — k8s 위의 cloud provisioning"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:28:00+09:00
tags: [devops, platform-engineering, crossplane, iac]
---

# Crossplane — k8s 위의 cloud provisioning

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. 무엇

- k8s API 로 cloud 자원 provisioning (AWS / GCP / Azure 등).
- Terraform 의 대안 + k8s native.
- CRD 로 추상화 → 개발자가 "database" 한 줄로 RDS 생성.

→ "infra-as-data" (manifest = state).

---

## 2. Terraform 과 차이

| | Terraform | Crossplane |
| --- | --- | --- |
| 모델 | declarative + state file | k8s CRD (controller-based) |
| state | 별도 backend (S3) | etcd (k8s) |
| reconcile | apply 시점만 | continuous (drift 자동 fix) |
| 추상화 | module | CompositeResource (★) |
| 통합 | 각 도구 | k8s 모든 도구 (RBAC / NetworkPolicy) |
| 사용 | 운영자 | 개발자 self-service |

→ **Terraform = ops tool, Crossplane = platform abstraction**.

---

## 3. 설치

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
    -n crossplane-system --create-namespace
```

```bash
# AWS provider
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata: {name: provider-aws}
spec:
  package: xpkg.upbound.io/upbound/provider-aws-rds:v1.0.0
EOF

# AWS credentials
kubectl create secret generic aws-creds \
    -n crossplane-system \
    --from-file=creds=./aws-credentials.txt
```

---

## 4. Managed Resource 예 (low-level)

```yaml
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata: {name: my-postgres}
spec:
  forProvider:
    region: ap-northeast-2
    engine: postgres
    engineVersion: "15.4"
    instanceClass: db.t3.micro
    allocatedStorage: 20
    dbName: mydb
    masterUsername: admin
    masterPasswordSecretRef:
      namespace: default
      name: db-password
      key: password
  writeConnectionSecretToRef:
    namespace: default
    name: postgres-conn
```

→ Crossplane controller 가 AWS API 호출 → RDS 생성.

→ secret 에 conn string 자동 저장.

---

## 5. Composite Resource (★ 추상화)

```yaml
# CompositeResourceDefinition — 새 API 정의
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: {name: xpostgres.db.example.com}
spec:
  group: db.example.com
  names: {kind: XPostgres, plural: xpostgres}
  claimNames: {kind: Postgres, plural: postgres}
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              properties:
                size: {type: string, enum: [small, medium, large]}
                region: {type: string}
```

```yaml
# Composition — 어떻게 만들 건지
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata: {name: postgres-aws}
spec:
  compositeTypeRef:
    apiVersion: db.example.com/v1alpha1
    kind: XPostgres
  resources:
    - name: rds
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            engineVersion: "15.4"
      patches:
        - fromFieldPath: spec.region
          toFieldPath: spec.forProvider.region
        - fromFieldPath: spec.size
          transforms:
            - type: map
              map: {small: db.t3.micro, medium: db.t3.small, large: db.t3.medium}
          toFieldPath: spec.forProvider.instanceClass
```

→ 개발자는 "Postgres" 만 알면 됨:

```yaml
apiVersion: db.example.com/v1alpha1
kind: Postgres
metadata: {name: my-db}
spec:
  size: small
  region: ap-northeast-2
  writeConnectionSecretToRef: {name: my-db-conn}
```

→ 자동 RDS 생성 + conn secret 발급.

---

## 6. golden path 예 (platform team 이 정의)

```
Platform 이 제공하는 abstraction:
  - Postgres (size + region → RDS)
  - Redis (size → ElastiCache)
  - Queue (→ SQS / SNS)
  - Bucket (→ S3 with encryption / versioning)
  - Service (→ k8s Deployment + Service + Ingress + monitoring)

개발자가 사용:
  apiVersion: platform.example.com/v1
  kind: Service
  metadata: {name: my-app}
  spec:
    image: ghcr.io/myorg/app:1.0
    database: {kind: Postgres, size: small}
    cache: true
    domain: my-app.example.com

→ 한 manifest 로 full stack.
```

---

## 7. drift detection (★)

```
누가 console 에서 RDS 수정 → Crossplane 이 자동 원복.
```

→ Terraform 은 manual `terraform plan` 필요. Crossplane = continuous.

---

## 8. Provider 종류

```
upbound/provider-aws-* (각 service 별)
upbound/provider-gcp-*
upbound/provider-azure-*
crossplane-contrib/provider-kubernetes
crossplane-contrib/provider-helm
crossplane-contrib/provider-github
```

→ 거의 모든 cloud + SaaS.

---

## 9. Backstage 와 통합

```
Backstage 의 Software Template:
  $ idp create "Postgres" --name my-db
  → Crossplane manifest 생성 + git commit
  → ArgoCD 가 apply
  → Crossplane controller 가 RDS 생성
  → Backstage 의 catalog 에 "Resource" entity 등장
```

→ self-service infra.

---

## 10. 대안

| | 무엇 |
| --- | --- |
| **Terraform** | 표준, state file, manual apply |
| **Pulumi** | 코드로 IaC (TypeScript/Python/Go) |
| **AWS CDK** | AWS 전용, 코드 |
| **AWS Service Catalog** | self-service portfolio |
| **kro / kratix** | k8s composition (Crossplane 경쟁) |

→ **개발자 self-service + drift 자동 fix = Crossplane** 강점.

---

## 11. 함정

1. **간단 case 도 Crossplane** — over-engineering. Terraform 으로 충분할 때.
2. **Composition 추상화 너무 일찍** — naive 부터 시작.
3. **state migration** — Terraform → Crossplane 어려움.
4. **delete propagation** — claim 삭제 시 진짜 cloud resource 도 삭제 (★ 주의).
5. **secret 관리 — Crossplane → k8s Secret** 평문. ESO / SealedSecret 와 같이.
6. **provider version drift**.

---

## 12. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[idp-concepts]]
- [[../iac/terraform-basics|↗ Terraform]]
- [[../iac/crossplane|↗ iac crossplane]]
