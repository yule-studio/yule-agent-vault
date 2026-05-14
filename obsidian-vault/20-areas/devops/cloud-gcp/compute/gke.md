---
title: "GCP GKE — Google Kubernetes Engine"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:15:00+09:00
tags:
  - gcp
  - compute
  - gke
  - kubernetes
---

# GCP GKE

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | GKE 개념 + 사용 |

**[[compute|↑ Compute]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

**가장 좋은 매니지드 K8s** (논쟁). Borg / Omega 의 후계인 Kubernetes 의 본가 Google 이 운영. AWS EKS / Azure AKS 보다 자동화 ↑.

---

## 2. 두 모드

| | Standard | Autopilot |
| --- | --- | --- |
| Node 관리 | 직접 (node pool) | GKE 가 자동 |
| 가격 | node 비용 + control plane | pod 단위 (vCPU/RAM 시간) |
| 유연성 | 높음 (custom daemonset 등) | 제약 |
| 권장 | 큰 / 특수 워크로드 | 대부분의 신규 (less ops) |

→ **Autopilot 권장** (단순). Standard = 큰 / 옛 / 특수.

---

## 3. 설치

### 3.1 Autopilot cluster

```bash
gcloud container clusters create-auto myapp \
  --region=asia-northeast3 \
  --release-channel=regular

gcloud container clusters get-credentials myapp --region=asia-northeast3
kubectl get nodes
```

### 3.2 Standard cluster + node pool

```bash
gcloud container clusters create myapp \
  --region=asia-northeast3 \
  --num-nodes=1 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling --min-nodes=1 --max-nodes=10 \
  --enable-autoupgrade \
  --enable-autorepair \
  --release-channel=regular \
  --workload-pool=PROJECT.svc.id.goog
```

### 3.3 Terraform (Autopilot)

```hcl
resource "google_container_cluster" "myapp" {
  name             = "myapp"
  location         = "asia-northeast3"
  enable_autopilot = true
  release_channel { channel = "REGULAR" }

  workload_identity_config {
    workload_pool = "${var.project}.svc.id.goog"
  }

  network    = google_compute_network.vpc.id
  subnetwork = google_compute_subnetwork.gke.id
}
```

---

## 4. Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    iam.gke.io/gcp-service-account: myapp@PROJECT.iam.gserviceaccount.com
```

→ pod 안 SDK 가 자동 GCP IAM 사용 (key 없음). AWS IRSA 와 같은 패턴.

---

## 5. Ingress + Load Balancer

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: "gce"
    networking.gke.io/managed-certificates: "myapp-cert"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: myapp
            port: { number: 80 }
```

→ GCP Cloud Load Balancing 자동 생성. **글로벌 LB** 가능 (multi-region).

또는 **Gateway API** (새 표준).

---

## 6. Cloud Logging / Monitoring 통합

기본 활성. log = `kubectl logs` + Cloud Logging (자동 indexing).

```bash
gcloud logging read 'resource.type="k8s_container"' --limit=50
```

---

## 7. 가격 (Seoul)

```
Autopilot:
  vCPU:   $0.0445 / vCPU·시간
  Memory: $0.0049 / GB·시간
  Storage:$0.0001 / GB·시간

Standard:
  Control plane (regional): $0.10 / 시간 = ~$73/월
  Node (GCE):               machine type 비용
  Free tier: $74.40 / month / cluster (control plane 1)
```

→ Autopilot 이 작은 클러스터에 더 쌈. 큰 = Standard.

---

## 8. 자동화 기능

- **Auto-upgrade** — Kubernetes / node 자동 업데이트
- **Auto-repair** — 망가진 node 재생성
- **Auto-scaling** — pod (HPA) + node (Cluster Autoscaler)
- **Auto-provisioning** — 새 node pool 자동 생성 (Standard)
- **Release Channels** — Rapid / Regular / Stable

---

## 9. AWS EKS vs GKE

| | EKS | GKE |
| --- | --- | --- |
| Control plane | $73/월 | $73/월 (Standard) — Autopilot 다른 모델 |
| node 자동화 | Karpenter (별도) | 내장 (auto-provisioning, repair) |
| 통합 | AWS | GCP |
| 학습 | 좋음 | 더 매끄러움 |

→ 양쪽 다 좋음. GCP 선택 시 GKE 자연.

---

## 10. 사용 시나리오

- microservice
- 데이터 파이프라인
- ML serving (GPU node)
- CI runner
- multi-tenant SaaS

부적합:
- 단순 응용 (Cloud Run 이 쉬움)
- 작은 정적 사이트 (Cloud Storage + Cloud CDN)

---

## 11. 함정

### 11.1 비용 최적화
node 비활용 → 큰 비용. cluster autoscaler + HPA + workload limit 정확히.

### 11.2 Workload Identity 누락
서비스 계정 key 사용 — 보안 위험.

### 11.3 release channel 의 자동 upgrade
production = Stable / Regular (Rapid 는 새 기능).

### 11.4 VPC IP 부족
default IP allocation. /20+ subnet 필요.

### 11.5 Autopilot 의 제약
DaemonSet 제한 / privileged container X / 일부 hostPath X.

### 11.6 LB 의 시간
첫 Ingress = 5-10 분 LB 프로비전.

---

## 12. 학습 자료

- GKE docs
- **Kubernetes 자체** — kubernetes.io
- **GKE Best Practices**

---

## 13. 관련

- [[compute]] — Compute hub
- [[gce]] — VM (옛 패턴)
- [[cloud-run]] — 더 단순
- [[../../kubernetes/kubernetes]] — K8s 일반
